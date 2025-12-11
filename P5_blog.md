# Practica 5: Laser Mapping
### Álvaro Quintana

En este proyecto, he desarrollado un **algoritmo de navegación** para la exploración y el mapeo de un almacén Se utiliza un **mapeo probabilístico** con una estrategia de **exploración por fronteras**.

---
<img width="837" height="265" alt="image" src="https://github.com/user-attachments/assets/540ead67-511a-4e50-aae0-033930c31ea3" />


## 1. Mapeado Robusto con Incertidumbre

El objetivo central es construir una representación interna fiel del entorno. Reconociendo que los sensores reales introducen **ruido e incertidumbre**, no podemos confiar en una única lectura para clasificar el espacio.

### Enfoque Bayesiano con Log-Odds

Para manejar esta incertidumbre, hemos implementado una **Rejilla de Ocupación**, una técnica que discretiza el entorno en celdas, donde cada una almacena la **probabilidad** de estar ocupada o libre. 

En lugar de operar con la multiplicación de probabilidades (la regla de bayes), utilizamos **Log-Odds**. Se transforma la regla de **actualización de Bayes** en una **suma aditiva**, lo que simplifica el código

* El mapa se inicializa en 0, lo que representa la **incertidumbre total** (probabilidad de ocupación $P=0.5$).
* Hemos definido constantes Log-Odds para la evidencia de **"Ocupado" (LO\_OCC)** y **"Libre" (LO\_FREE)**, basadas en probabilidades iniciales de alta confianza ($P=0.7$ y $P=0.3$), ya que el laser es un sensor fiable (no como,por ejemplo, el ultrasonidos).
* Se aplica una **saturación (`clamp`)** a los valores de Log-Odds para evitar que la confianza crezca indefinidamente, asegurando que el mapa pueda **adaptarse a cambios dinámicos** en el entorno (ej., objetos que se mueven).

### Modelo del Sensor y Trazado de Rayos

Para integrar la información del sensor láser en el mapa:

1.  Se utiliza una técnica de **Trazado de Rayos (`Ray Tracing`)** (algoritmo de Bresenham) para identificar todas las celdas atravesadas por el haz del láser.
2.  Las celdas **atravesadas** son penalizadas (se les resta el valor $\text{LO}_{\text{FREE}}$) para indicar que no está ocupado.
3.  Si el láser detecta un obstáculo dentro de su rango útil, la celda final de impacto es recompensada (se le suma el valor $\text{LO}_{\text{OCC}}$), diciendo que está ocupada.

Este proceso iterativo permite que el **ruido del sensor se filtre** y que las representaciones de obstáculos y espacio libre adquieran una **alta fidelidad** tras múltiples observaciones.

---

## 2. Estrategia de Navegación: Exploración de Fronteras

Para lograr una exploración inteligente, la solución implementa la estrategia de **Exploración de Fronteras**.

### Capa de Abstracción y Planificación

Para que el robot pueda tomar decisiones, se crea una capa de abstracción (Grid de navegación):

1.  **Grid de Navegación:** El mapa de Log-Odds detallado se reduce a una **rejilla de menor resolución que el grid de ocupación** un poco más amplio que lo que ocupa nuestro robot. Esto hace que los calculos sean más ágiles y tengamos un espacio de seguridad para evitar colisiones.
  
3.  **Clasificación de Estados:** Cada macro-celda del Grid de Navegación se clasifica en uno de tres estados **Obstáculo**, **Espacio Libre** o **Desconocido**, basándose en la cantidad de evidencia del mapa de Log-Odds
### Detección de Fronteras y BFS

El núcleo de la autonomía reside en la detección de **fronteras**: la línea divisoria entre el espacio conocido (clasificado como **Libre**) y el territorio inexplorado (clasificado como **Desconocido**). 

Para alcanzar estas zonas de manera óptima, he implementado un algoritmo de **Búsqueda en Anchura (BFS)**.

* El BFS garantiza encontrar la **frontera accesible más cercana** al robot.
* Esto asegura una expansión sistemática del mapa, optimizando el tiempo de exploración y evitando desplazamientos largos e ineficientes.

---

## 3. Navegación local

Una vez que el algoritmo de BFS identifica la celda de la frontera más cercana, el movimiento se ejecuta a través de un control proporcial del error.
Se emplean dos controladores **Proporcionales** para comandar la velocidad lineal y angular del robot.
# [RESULTADO](https://www.dropbox.com/scl/fi/pb30qqw8k4opckqup3wx4/Laser_mapping-2025-12-11_14.15.49.mp4?rlkey=ike7j82ijrb5e6wo11riglk0n&st=8kc6uk9j&dl=0)
