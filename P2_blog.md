# Practica 2: Rescue People

El problema lo he dividido en las siguientes etapas:
  1. Trabajar con coordenadas UTM: Conocer donde se encuentra la base y los supervivientes, en un sistema en el que pueda trabajar
  2. Planificación y ejecución de una ruta de barrido desde el punto de busqueda de los supervivientes
  3. Detección de rostros usando Haar Cascades de open cv
  4. Hacer única la detección de cada superviviente
  5. Simular la batería
Una vez funcionan los apartados anteriores, implemento la lógica de la aplicación utilizando una máquina de estados finito con forma:
<p align="center">
<img width="300" alt="image" src="https://github.com/user-attachments/assets/4bf151ff-2c8f-4472-a7bf-c45dab534a77" />
</p>

## 1.UTM a Mundo Gazebo
Si utilizamos una calculadora de coordenadas utm en los puntos dados por enunciado notamos que Barco (40º16’48.2” N, 3º49’03.5” W.) y Supervivientes (40º16’47.23” N, 3º49’01.78” W.) Gracias a esta cercanía, es posible aplicar una aproximación de Tierra plana, lo que simplifica los cálculos de conversión entre coordenadas geográficas y coordenadas cartesianas del mundo en Gazebo.
Aunque esta aproximación introduce un pequeño error, su impacto es despreciable en distancias cortas y dentro de un entorno simulado. La conversión se realiza calculando la diferencia entre las coordenadas del barco y las de los supervivientes, expresando el desplazamiento en metros hacia el norte (latitud) y el este (longitud). Esta función devuelve el offset (desplazamiento aproximado en metros) entre el barco y los supervivientes, que se utiliza posteriormente para definir el punto de aproximación inicial del dron dentro del entorno de Gazebo.

## 2. Planificación y ejecución de una ruta de búsqueda
Para esta parte he decidio utilizar el punto de aproximación como centro de la búsqueda, que se hará en cuadrados en cada uno de los cuadrantes correspondientes
La trayectoria se define a partir de:

* Un vértice inicial (init_vertex), que representa el punto donde el dron comienza la búsqueda.
* Las dimensiones del área (base y height) y el paso (step) entre franjas de vuelo.
* Los cuadrantes cartesianos donde se ejecutará la misión

El resultado es una cola de puntos objetivo (deque), que el dron recorrerá de forma ordenada para cubrir el área. En esta parte me ha sido muy util hacer pruebas con el módulo turtle de python:
<p align="center">
  <img src="https://github.com/aquintan4/robotica_servicio_aquintan4/blob/main/resources/square_path.gif?raw=true" width="500" alt="Square Path Simulation">
</p>

Para el video se muestra la búsqueda en los cuadrantes 2 y 4 por simplicidad, aunque en una operación real utilizaria los 4.

## 3. Detección de rostros usando Haar Cascades de open cv

La detección de posibles supervivientes se realiza con un clasificador Haar Cascade sobre la cámara ventral:

1) **Filtrado previo del mar (HSV)**  
   - Convertimos a HSV y enmascaramos azules (`LOWER_BLUE`, `UPPER_BLUE`), dejando solo zonas con información útil.
   - Si la máscara tiene menos de `MIN_PIXELS`, se omite la detección para ahorrar cómputo.

2) **Preprocesado**  
   - Pasamos a escala de grises y aplicamos `equalizeHist` para mejorar contraste.

3) **Robustez a orientación**  
   - Rotamos la imagen en pasos de `rotation` grados y ejecutamos `detectMultiScale` en cada rotación.  
   - Reproyectamos el centro del bounding box a la imagen original.

4) **Supresión de duplicados (misma imagen)**  
   - Evitamos contar la misma cara varias veces comparando centros con un umbral proporcional al tamaño del ROI

5) **Pixel → metros (marco del dron)**  
   - Con intrínsecos aproximados (HFOV~60°): `fx = fy = 554.26`, `cx = cols/2`, `cy = rows/2`.  
   - Proyectamos el centro `(u,v)` a desplazamiento métrico:  
     `x_cam = (u - cx) * z / fx`, `y_cam = (v - cy) * z / fy`.

6) **Cámara → mundo (plano z=0)**  
   - Rotamos por `yaw` y sumamos posición del dron: `dron_relative_2_world(...)`.  
   - Deducimos duplicados en mundo con umbral `NEW_SURVIVOR_DISTANCE` y guardamos ubicaciones nuevas.

## 4. Hacer única la detección de cada superviviente

En esta parte el objetivo era evitar que el dron contara varias veces al mismo superviviente cuando lo detectaba desde distintos ángulos o posiciones.
Inicialmente intenté hacerlo utilizando las regiones de interés (ROIs) de las caras detectadas. Cada vez que el Haar Cascade encontraba un rostro, extraía esa zona de la imagen y la comparaba con las que ya tenía almacenadas, usando una medida de similitud con cv.matchTemplate. Si el nuevo recorte se parecía lo suficiente a uno anterior, se consideraba la misma persona; si no, se añadía al registro.

Aunque la idea era buena sobre el papel, en la práctica resultó poco robusta. Las detecciones variaban mucho por la iluminación, la altura del dron o el ángulo de visión, y eso hacía que el mismo rostro se registrara múltiples veces. Además, el proceso era pesado, ya que implicaba rotar la imagen, detectar y comparar plantillas en cada iteración, lo que reducía notablemente la eficiencia. En la mayoría de los casos, terminaba acumulando muchísimos falsos positivos y duplicados.

Por eso acabé cambiando el enfoque a algo más simple y estable: basar la unicidad en la posición. En lugar de comparar imágenes, lo que hago es registrar las coordenadas de cada nuevo superviviente en el mundo y comprobar si ya existe otro punto cercano (dentro de una distancia definida, por ejemplo 3.5 metros). Si no hay ninguno, se guarda como nuevo; si ya hay uno próximo, se considera la misma persona. Este método, aunque menos sofisticado visualmente, es mucho más fiable en simulación, porque aprovecha la información de posición y orientación del dron, que es estable y precisa.


