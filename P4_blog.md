# PRACTICA 4: Amazon Warehouse  
### Álvaro Quintana  
---

<img width="500" alt="image" src="https://github.com/user-attachments/assets/994ae623-87cd-4e43-bc45-4ea53aaee9ad" />

Esta práctica va de aprender a usar **OMPL** para hacer path planning dentro de un mapa tipo almacén de Amazon. El dibujo del enunciado ya deja bastante claro lo que se pretende:

<img width="500" alt="image" src="https://github.com/user-attachments/assets/a6908931-fa9f-473d-ab99-dc9f4a6218ee" />

---

## 1. Ajustar coordenadas: Mundo ↔ Mapa

Antes de hacer nada hay que tener el mundo y el mapa sincronizados. Igual que en la P1, cojo varias posiciones reales con `HAL.getPose3d()` y busco dónde caen en el mapa en píxeles.

Una cosa que he comprobado en esta práctica:  
**no importa tener muchos puntos, importa tenerlos buenos y bien repartidos**.

Con esos puntos monto un sistema por mínimos cuadrados y me saca la matriz que transforma mundo → mapa. Invierto esa matriz y ya tengo mapa → mundo.

---

## 2. Espacio de estados

El robot se define por:  
- `x`  
- `y`  
- `theta`

Así que OMPL trabaja en **R²**, porque `theta` es parte de la configuración y ya está incluida.  

Los límites los pongo igual que el tamaño del mapa, porque fuera de ahí no hay nada de interes.
---

## 3. Validador de estados

Aquí viene la parte crítica de la práctica:

### Footprint del robot
Cada robot (holonómico o Ackerman) lo represento como un rectángulo binario con su alto × ancho.  
Después lo roto según su `yaw` usando OpenCV. Hay que calcular bien la caja nueva o se corte la figura.

### Comprobación en el mapa
Alineo ese footprint rotado sobre la posición del robot en el mapa.  
Si algún píxel del footprint pisa un píxel que no sea libre (127) → **estado inválido**.
Si aquí algo fallara, OMPL te genera rutas invalidas.
---

## 4. Planificación con OMPL

Ya con espacio de estados, límites y validador, se monta el problema.

En mi caso uso:
- **Reeds-Shepp** (porque Ackerman no gira sobre sí mismo)
- **RRT\*** (más suave que RRTConnect, aunque tarde algo más)

Cuando OMPL devuelve una ruta, la **interpolé** para meter más puntos.  
Para holonómico no marca la diferencia, pero para Ackerman es necesario suavizar para evitar giros que físicamente no puede hacer.

---

## 5. Control para seguir la ruta

Aquí simplemente calculo:
- dirección hacia el siguiente punto  
- error angular  
- ajusto `W` proporcionalmente  
- decido dirección (delante/detrás) según el error  
- y paso al siguiente punto cuando estoy cerca
---

## 6. Resultado

Con todo esto implemento una FSM que:

1. Planificar hasta la estantería  
2. Llegar sin chocar  
3. “Eliminar” la estantería del mapa  
4. Cargarla  
5. Replanificar de vuelta  
6. Dejarla donde toca  

Todo respetando las limitaciones del Ackerman.

# [resultado 1](https://www.dropbox.com/scl/fi/iw71jueu4xzp091mthiym/ACKERMAN.webm?rlkey=zz7oyyczgapvakv15a3yrlz3y&st=w0w206p2&dl=0)
# [resultado 2](https://www.dropbox.com/scl/fi/iw71jueu4xzp091mthiym/ACKERMAN.webm?rlkey=zz7oyyczgapvakv15a3yrlz3y&st=bbf3dll4&dl=0)
