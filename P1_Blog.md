# Practica 1: Algoritmo BSA de cobertura

### 1. Matriz de transformación de coordenadas
La primera etapa consiste en hallar una relación entre las coordenadas del mundo (obtenida mediante la función HAL.getPose3d) y las coordenadas del mapa (una matriz de píxeles NxM de X canales).
Sin esta etapa, por muy buena planificación que hicieramos sobre el mapa de ocupación; no sabríamos cual es el punto de comienzo, donde nos encontramos en cada momento, o si estamos siguiendo correctamente el camino calculado.
Para ello he utilizado una matriz de transformación de proyección 3d (mundo de gazebo) a 2d (mapa de ocupación), con la forma:<br>
<img width="35%" alt="image" src="https://github.com/user-attachments/assets/6cd68fa6-e538-48d0-8c3e-0850cde74bc7" />

Donde nuestras incógnitas serán **tx ty theta scale**. Las incoginitas, son la medida de cuanto debemos transformar (desplazar, rotar y ampliar) las coordenadas de nuestro mundo, para obtener
esa misma coordenada en el mapa. Debemos colocar a nuestro robot en puntos clave del mapa y tratar con la mayor precisión posible encontrar el píxel correspondiente en el mapa. Para esta última tarea
he utilizado el siguiente script.

```python3
import cv2
import matplotlib.pyplot as plt
from matplotlib.patches import Rectangle

img = cv2.imread("mapgrannyannie.png")
if img is None:
    raise FileNotFoundError("No se encontró 'mapgrannyannie.png'.")

img_rgb = cv2.cvtColor(img, cv2.COLOR_BGR2RGB)
h, w = img_rgb.shape[:2]


raw = input("Introduce coordenadas x y: ").strip().replace(",", " ")
parts = [p for p in raw.split() if p]
if len(parts) != 2:
    raise ValueError("Debes introducir exactamente dos números: x y")

x, y = map(int, parts)

if not (0 <= x < w and 0 <= y < h):
    raise ValueError(f"Coordenadas fuera de la imagen. Rango x=[0,{w-1}], y=[0,{h-1}]")

fig, ax = plt.subplots()
ax.imshow(img_rgb, origin='upper')

ax.plot(x, y, marker='o', markersize=4, color='red')

rect = Rectangle((x - 0.5, y - 0.5), 1, 1, linewidth=1.5, edgecolor='red', facecolor='none')
ax.add_patch(rect)

ax.annotate(
    f"({x}, {y})",
    xy=(x, y), xycoords='data',
    xytext=(x + 30, y - 30), textcoords='data',
    arrowprops=dict(arrowstyle="->", lw=1.5, color='red'),
    color='red', fontsize=9, bbox=dict(boxstyle="round,pad=0.2", fc="white", ec="red", lw=0.8)
)

ax.set_title("píxel marcado")
ax.set_xlim(-0.5, w - 0.5)
ax.set_ylim(h - 0.5, -0.5)
plt.tight_layout()
plt.show()
```
Ejemplo de ejecución:<br>
<img width="35%" alt="Figure_1" src="https://github.com/user-attachments/assets/6b1c6173-2127-4db8-a29d-527056c60656" />

Al tener 4 incógnitas deberíamos tener al menos 4 ecuaciones, es decir con una correspondencia de 2 puntos en mundo y en mapa podríamos encontrar las incognitas. Sin embargo no bastará, debemos relacionar
cuantos más puntos mejor, ya que en todos ellos habrá algún tipo de ruido y cuantos más tengamos más reduciremos el error en la transformación.

Notamos que para obtener nuestras incógnitas debemos realizar una optimización de un sistema compatible indeterminado, en mi caso utilizaré la optimización por mínimos cuadrados, concretamente la implementación
de la biblioteca numpy.

### 2. Dilatación de los obstáculos
Este paso es importante para hacer que nuestro robot no se choque con ningún obstáculo a la hora de hacer la ruta, por lo tanto debemos expandirlo el radio de nuestro robot más una distancia de seguridad. En mi caso he utilizado:
```python3
_, bin_map = cv.threshold(img, 1, 255, cv.THRESH_BINARY)
dist = cv.distanceTransform(bin_map, distanceType=cv.DIST_L2, maskSize=5)
R = radio_del_robot_en_px + margen_de_seguridad_en_px
inflated_obstacles = (dist <= R).astype(np.uint8) * 255
inflated_map = np.where(inflated_obstacles > 0, 0, 255).astype(np.uint8)
```
1. Usamos distanceTransform para calcular en cada píxel de espacio libre, la distancia euclídea (L2) al obstáculo más cercano (un píxel con valor 0).
2. Marcamos como obstáculo inflado todo píxel de espacio libre cuya distancia al obstáculo real sea ≤ R.
3. Construimos el mapa inflado final: los píxeles marcados en inflated_obstacles pasan a 0 (ocupado); el resto a 255 (libre)

Esta transformación a priori resulta más extraña que usar directamente una función para dilatar, o pasar un kernel de forma manual por la imagen. Sin embargo he probado todas estas transformaciones y el resultado es muy asimetrico, ya que hay zonas que están extremadamente dilatadas y zonas que no se han dilatado lo suficiente.

### 3. Obtener mapa de celdillas a partir de nuestro mapa con los obstáculos dilatados
Esta etapa consiste en definir el mapa de obstaculos en celdillas en el que aplicaremos el algoritmo de BSA. El tamaño de las celdillas debe de ser igual al diametro de nuestro robot y menos una distancia pequeña, de forma que aunque pasemos por algunos puntos varias veces, aseguramos que no haya huecos entre celdillas (cosa que ocurriría si la celdilla fuera más grande que el díametro del robot).

Para obtener este mapa de celdillas lo primero que hago es añadir filas y columnas a nuestra matriz de forma que se pueda usar el kernel con el tamaño de celdilla ya mencionado. Luego iteraremos sobre este mapa dilatado con nuestro kernel e iremos construyendo una matriz más pequeña en la que marcaremos cada celdilla como ocupada si en la posición del kernel hay algun obtaculo y libre si en este no hay ninguno.

### 4. BSA
Sobre la matriz calculada en el paso anterior utilizo el algoritmo BSA, para barrer toda la superficie posible. Esta parte consiste en: dado el mapa de celdillas y la posición inicial del robot.
1. Voy marcando las celdillas vecinas libres respecto a la celdilla actual como puntos de retorno si estos están libres; los ignoro en caso de estar ocupada o de ya estar visitada.
3. De todos los puntos de retorno elijo una dirección para avanzar y me ziño a ella (realizando el paso 1 en cada paso) hasta que me encuentre con un obstáculo o una celdilla visitada, entonces cambio a la sigiente dirección disponible
4.  Continuo con los pasos anteriores hasta finalmente no tener disponible ninguna de las direcciónes. En este momento se marca como punto crítico ese punto y se lanza una búsqueda en anchura (BFS) para encontrar el punto de retorno más cercano.
5. Se itera sobre estos pasos hasta que no encuentre ningún punto de retorno.

Esta parte ha sido bastante engañosa a la hora de programar, ya que he comenzado utilizando espirales, sin embargo a la hora de visualizar como se está calculando el camino paso a paso y visualizando no me parecía la forma más eficiente. Por lo tanto cambié completamente el enfoque e hice que el robot avanzara por prioridad de direcciones de forma que siempre que pueda ir a la dirección más prioritaria, en caso de no poder utilizaría la segunda dirección más prioritaria y así respectivamente. Esto se traduce en unos barridos sistematicos de derecha a izquierda (o arriba a abajo según las prioridades elegidas) de forma que se minimizan bastante los puntos críticos y la expansión en el visualizar parece bastante eficiente.

Sin embargo, el verdadero cuello de botella ocurre en el control del robot, concretamente en los giros. Entonces esta forma de barrer celdillas reduce significativamente los puntos críticos pero añade una cantidad de giros mucho mayor, haciendo que haya muchas detenciones para rectificar la dirección y se traduzca en un pilotaje mucho más lento y tedioso. Por lo que finalmente decidí volver a las espirales.

### 5. Pilotaje reactivo
He tratado de reducir al mínimo las oscilaciones para asegurarme que viajo lo más recto posible a traves de la celdillas, esto se ha traducido en dos PDs para la velocidad angular, uno cuando me encuentro desorientado respecto al objetivo (en la que la velocidad lineal es 0) y otro cuando me encuentro orientado respecto al punto objetivo (en el que la velocidad lineal depende de la distancia al objetivo y la orientación a este).
Sin embargo por más que ajustara estos PDs o la lógica seguía enfrentandome a un gran problema de oscilaciones. Que se traducía en un zig-zag constante, en la que la reorientación no solo "ocurría en las esquinas de las espirales" si no en mitad de trayectorias rectas.
La solución implementada consiste en iterar sobre la cola de objetivos desde el objetivo actual hasta identificar el siguiente punto que no se encuentre alineado con la trayectoria.

De este modo, se descartan los objetivos intermedios redundantes, permitiendo que el robot se desplace en línea recta entre los puntos relevantes siempre que el control de movimiento sea suficientemente preciso.Esta estrategia contribuye a reducir el efecto del ruido en el pilotaje y a evitar trayectorias en zigzag, logrando un desplazamiento más fluido y eficiente.

[Ver resultado](https://www.dropbox.com/scl/fi/nr7f3mcp001b5icgbgsab/p1_servicio_final.mp4?dl=0&rlkey=q45lztcwn7yqhw07fb3bumeau&st=x7nzt8p3)



