# Practica 3
### Álvaro Quintana
---
<img width="299" height="168" alt="image" src="https://github.com/user-attachments/assets/324662ca-147d-48d2-b1ef-be0b25076b15" />

La práctica es totalmente dependiente del uso del láser, debemos interpretar correctamente los datos del sensor para que la aplicación funcione lo mejor posible.
Primeramente debemos entender cómo trabaja el láser: este recoge información de n_medidas, en nuestro caso 180 medidas, para cada ángulo entre minAngle y maxAngle.
Cada salto entre medidas viene dado por:
```python3
step = (maxAngle - minAngle) / n_medidas
```
De esta forma, se ve fácilmente que un índice cualquiera de nuestro array representa un ángulo:
```
angle_i = min_angle + i * step
```

Es decir, saltamos i pasos desde nuestro ángulo inicial.
El caso es que las medidas de los extremos (mínimo y máximo) no son tan interesantes como el rayo que sale perpendicular al láser (el que apunta recto hacia adelante). Sería más cómodo interpretar los datos si ese rayo representara el ángulo 0. En otras palabras, el ángulo 0 debería ser el punto medio entre el haz mínimo y el máximo, ya que de esta forma solo con el signo podemos saber si estamos trabajando con la derecha o con la izquierda.
Para conseguirlo, desplazamos el ángulo original restando el offset correspondiente a la mitad del rango angular. Así podemos saber fácilmente si una medida está en el centro, a la derecha o a la izquierda, simplemente mirando el signo del ángulo o si este vale 0.

### Filtrado de medidas
Otra parte importante es filtrar los valores del láser para poder operar con ellos de forma segura.
Si tenemos valores infinitos o que estén fuera del rango medible, los sustituyo por 0 para poder ignorarlos fácilmente después. Aunque a primera vista un 0 podría parecer una distancia válida (muy corta, por ejemplo en caso de colisión), en realidad no lo es, porque siempre será menor que el laser_min. De esta forma, 0 pasa a significar “valor no válido”.

El comportamiento está organizado en una máquina de estados (FSM). Cubre desde buscar hueco, posicionarse, maniobra en reversa, ajustes y final. No hace falta que me pares a las constantes: la FSM controla la secuencia lógica y los cambios de estado se hacen con las comprobaciones que verás a continuación.

### Maniobra de aparcamiento

Una vez tengo los datos del láser listos, el siguiente paso es encontrar un hueco para aparcar.
Toda la lógica está organizada con una pequeña FSM que va pasando por varios estados: buscar hueco, colocarse, girar en reversa y ajustar.

Para seguir la línea de coches utilizo el láser frontal. Probé con el de la derecha, pero la información llega demasiado tarde, así que el frontal me da lecturas más coherentes con la dirección del coche y me deja anticipar mejor en el PD.
Utilizo un rango de ángulos del lado que me interesa y me quedo con las distancias válidas.
De cada punto obtengo su componente lateral (distancia real hacia el lado), y hago una media ponderada.
Para quedarme con el borde “real” de la hilera de coches y evitar los valores que se salgan.
El peso de cada punto depende de lo parecidos que sean entre sí, así que los puntos que representan la misma pared o coche tienen más influencia en el resultado.
Esa media ponderada es la distancia que uso como referencia para mantenerme paralelo a la fila.

El control lo hago con un PD bastante básico. La referencia es la distancia lateral deseada y con eso ajusto la velocidad angular (W) mientras mantengo la velocidad lineal fija.
Con esto el coche se mantiene siguiendo los coches aparcados con bastante estabilidad.

Para detectar un hueco, ahora sí, uso el láser derecho. Miro un rango concreto de ángulos y calculo el porcentaje de medidas que valen cero, que en mi caso representa los puntos donde el láser no ve nada (distancias infinitas).
Si el porcentaje es bastante alto, significa que ahí hay espacio libre.
Cuando eso ocurre paso al siguiente estado, donde el coche avanza un poco más hasta colocarse con la parte trasera alineada con el coche siguiente, para tener espacio suficiente al girar después.

A partir de ahí empiezo la maniobra de aparcar. Para girar no me baso en las lecturas del láser porque no encontraba las referencias adecuadas en los tres láseres. En su lugar uso el yaw para ver cuánto he girado. Cuando alcanzo el ángulo que me interesa, paro el giro y cambio de sentido.

Mientras retrocedo, uso el láser trasero para controlar la distancia con el coche de atrás.
Voy vigilando si la distancia mínima baja de la de seguridad o si ya no tengo suficientes lecturas válidas que indiquen que el coche está alineado.

Después vienen las fases de rectificación, tanto hacia adelante como hacia atrás.
Aquí lo que hago es mover el coche mientras miro un rango pequeño del láser frontal (o trasero).
Si en ese rango hay suficientes valores válidos, significa que ya estoy dentro y alineado
Si detecto que estoy demasiado cerca de algo, cambio de sentido y repito hasta quedar centrado.

# [RESULTADO](https://www.dropbox.com/scl/fi/d6o973yi715a4dg7f7ite/P3_video-2025-11-07_21.22.13.mp4?rlkey=i01df5yc50xstszldbxcx3bpz&st=zrznqoxy&dl=0)
