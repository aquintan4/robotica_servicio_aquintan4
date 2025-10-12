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
