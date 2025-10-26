# Practica 2: Rescue People

El problema lo he dividido en las siguientes etapas:
  1. Trabajar con coordenadas UTM: Conocer donde se encuentra la base y los supervivientes, en un sistema en el que pueda trabajar
  2. Planificación y ejecución de una ruta de barrido desde el punto de busqueda de los supervivientes
  3. Detección de rostros usando Haar Cascades de open cv
  4. Hacer única la detección de cada superviviente
  5. Simular la batería
Una vez funcionan los apartados anteriores, implemento la lógica de la aplicación utilizando una máquina de estados finito con forma:

# 1.UTM a Mundo Gazebo
Si utilizamos una calculadora de coordenadas utm en los puntos dados por enunciado notamos que Barco (40º16’48.2” N, 3º49’03.5” W.) y Supervivientes (40º16’47.23” N, 3º49’01.78” W.) Gracias a esta cercanía, es posible aplicar una aproximación de Tierra plana, lo que simplifica los cálculos de conversión entre coordenadas geográficas y coordenadas cartesianas del mundo en Gazebo.
Aunque esta aproximación introduce un pequeño error, su impacto es despreciable en distancias cortas y dentro de un entorno simulado. La conversión se realiza calculando la diferencia entre las coordenadas del barco y las de los supervivientes, expresando el desplazamiento en metros hacia el norte (latitud) y el este (longitud). Esta función devuelve el offset (desplazamiento aproximado en metros) entre el barco y los supervivientes, que se utiliza posteriormente para definir el punto de aproximación inicial del dron dentro del entorno de Gazebo.

# 2. Planificación y ejecución de una ruta de búsqueda
Para esta parte he decidio utilizar el punto de aproximación como centro de la búsqueda, que se hará en cuadrados en cada uno de los cuadrantes correspondientes
La trayectoria se define a partir de:

* Un vértice inicial (init_vertex), que representa el punto donde el dron comienza la búsqueda.
* Las dimensiones del área (base y height) y el paso (step) entre franjas de vuelo.
* Los cuadrantes cartesianos donde se ejecutará la misión

El resultado es una cola de puntos objetivo (deque), que el dron recorrerá de forma ordenada para cubrir el área. En esta parte me ha sido muy util hacer pruebas con el módulo turtle de python:


