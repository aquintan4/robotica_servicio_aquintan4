# Práctica 3
# Álvaro Quintana
La práctica es totalmente dependiente del uso del laser, debemos interpretar correctamente los datos del laser para hacer la aplicación lo mejor posible. Primeramente debemos entender como funciona el laser, este recoge información de n_medidas,
en nuestro caso son 180 medidas, para cada ángulo desde minAngle a maxAngle, y cada salto entre medidas está dado por:

'''
step = (max - min) / n_medidas, 
'''

de está forma vemos facilmente que un indice cualquiera de nuestro arrays representa el ángulo:
'''
angle i = min_angle + i * step, 
'''

Es decir saltamos i pasos desde nuestro ángulo inicial, el caso es que las medidas de los extremos min, max no son tan interesantes como el rayo que sale perpendicular al laser (el que va más recto), nos sería más facil interpretar los datos si
este rayo representara el ángulo 0 (Es decir el angulo 0 debería ser el del medio entre nuestro haz de luz mínimo y máximo). Para ello deberíamos desplazar el angulo 0 original, esta medida a la izquierda. Numéricamente sería restando este offset
de la mitad del angulo entre el mínimo y el máximo, de esta forma sabemos si las medidas están en el centro, derecha o izquierda simplemente comprobando el signo del angulo o viendo si este es 0.

Lo siguiente que quiero hacer es encontrar un sitio en el que aparcar:
* Lo primero que necesitamos saber es si vamos a aparcar a la izquierda o a la derecha que en nuestro entorno simulado dependerá de si estamos subiendo o bajando la calle. Saber el lugar es necesario para arrimarnos a ese lateral y evitar que el coche
quede fuera del sitio






