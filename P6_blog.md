# Practica 6: Localización Visual con AprilTags
## Álvaro Quintana

<img width="200" alt="image" src="https://github.com/user-attachments/assets/491628f1-78ad-48bc-84e1-25308e2d09bf" />

El objetivo es dar al robot la capacidad localizarse en un espacio 2D. El sistema permite al robot estimar su pose (posición (x, y) y orientación yaw) en todo momento.

Para lograrlo, la solución se utiliza tanto la odometría de las ruedas (que acumula error con el tiempo), como la localización mediante visión, con **AprilTags** repartidos en el entorno.

## 1. Calibración Extrínseca
Antes del procesamiento de imagen, se establece la relación entre el sensor y el robot, lo he calculado mirando las posiciones de los frames del robot en el archivo sdf del enunciado:
* **Matriz `robot2camera`**: Matriz de transformación homogénea que representa la posición física de la cámara respecto al centro del robot. Es necesario porque la cámara se encuentra desplazada (hacia adelante y arriba)
*  y sus ejes de coordenadas difieren de los del robot (el eje Z de la cámara apunta al frente, mientras que en el robot corresponde al eje X).
* **Matriz `camera2robot`**: Inversa de la matriz anterior. Esta matriz es la encargada de traducir cualquier detección realizada en el marco de referencia de la cámara al sistema de coordenadas del centro del robot.

## 2. Localización Visual

La función principal, `update_visual_localization`, ejecuta la lógica de autolocalización:

### A. Del 2D al 3D: Resolviendo el problema PnP
Cuando el detector `pyapriltags` identifica un marcador, obtenemos coordenadas en píxeles. Para relacionar estos puntos con el espacio 3D, se aplica el algoritmo **Perspective-n-Point (PnP)** mediante `cv2.solvePnP`.

Las matrices utilizadas son:
1.  **Modelo del objeto (`TAG_LOCAL_CORNERS`)**: Coordenadas 3D reales de las esquinas del marcador (de tamaño conocido) restando la mitad del ancho y alto para obtener las esquinas respecto al centro.
2.  **Observación (`img_pts`)**: Puntos detectados en la imagen actual.
3.  **Cámara (`matrix_camera`)**: Matriz de calibración intrínseca (distancia focal y centro óptico dada por el enunciado). La he utilizado aunque he notado que es una aproximación muy grande, con una calibración más precisa el resultado mejoraría notablemente.

El resultado son los vectores de rotación y traslación (`rvec` y `tvec`) que definen la posición del tag desde la cámara ($T_{cam\_tag}$).

### B. Cadena de Transformaciones

El objetivo final es actualizar las variables globales `est_x`, `est_y` y `est_yaw` en el mundo. Para ello, se realiza una multiplicación de matrices en cadena para transitar entre los sistemas de coordenadas:

1.  Se obtiene la posición absoluta del marcador en el mapa ($T_{world\_tag}$) desde la configuración.
2.  Se invierte la transformación PnP para obtener la cámara respecto al marcador ($T_{tag\_cam}$).
3.  Se aplica la transformación final:

$$Pose_{Robot} = T_{Mundo\_Tag} \times T_{Tag\_Camara} \times T_{Camara\_Robot}$$

En el código, esto se implementa como:

```python
T_world_robot = T_world_tag @ T_tag_cam @ camera2robot
```
Al extraer la traslación y rotación de la matriz resultante, se obtiene una posición bastante exacta del robot y se actualiza la estimación. Se podría mejorar, como hemos mencionado, con una mejor calibración (mejor matriz
de parámetros intrínsecos)

## 2. Mantenimiento de Posición con Odometría

Dado que la cámara tiene un campo de visión limitado, se implementa un sistema de respaldo para cuando no hay marcadores visibles:<br> 

1. Se leen los encoders mediante HAL.getOdom()
2. Se calcula el diferencial (delta de x, y, yaw) respecto a la iteración anterior.
3. Este delta se suma a la última estimación conocida.

Aunque este método acumula ruido también genera continuidad del movimiento. En el momento en que se detecta un nuevo AprilTag, la localización visual sobrescribe la estimación, corrigiendo el error acumulado.

## 3. Estrategia de Navegación
Para validar la localización, el robot utiliza una máquina de estados finitos que implementa un bump&go para la exploración:<br>
FORWARD: El robot avanza buscando marcadores.<br>
BACK y TURN: Si el láser detecta un obstáculo a menos de la distancia de seguridad (OBSTACLE_DIST, 0.4 metros), el robot retrocede y gira aleatoriamente.<br>


# [RESULTADO](https://www.dropbox.com/scl/fi/64pv0sp056hwdg4sjn72i/visual_loc-2025-12-31_10.58.32.mp4?rlkey=1m6h2bvds4ijdww1hm0mshah7&st=350b5l3p&dl=0)


