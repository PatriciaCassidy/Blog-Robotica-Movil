# Práctica 4: Navegación Global con Gradient Path Planning (GPP)

## 1. Introducción

En esta práctica el objetivo es programar un coche autónomo (un taxi) capaz de navegar por una ciudad desde su posición actual hasta un destino remoto seleccionado por el usuario. A diferencia de las prácticas anteriores, en este ejercicio el coche dispone de un sensor GPS y de un mapa estático de la ciudad, pero **no dispone de sensor láser**.

La finalidad principal es implementar el algoritmo de planificación global **Gradient Path Planning (GPP)**. Este algoritmo funciona como un "plan como recurso", generando un campo escalar de potenciales. El destino actúa como un pozo de atracción y los obstáculos como barreras. El coche debe seguir el gradiente descendente de este campo para trazar la ruta más corta y segura, gobernando su velocidad lineal (V) y angular (W) mediante un bucle reactivo. Además, el sistema debe esperar a un nuevo destino cuando el usuario haga clic en otro punto del mapa.

## 2. Planteamiento del problema

El desarrollo se dividió en tres grandes bloques: el procesamiento del mapa, la generación del campo GPP y el pilotaje reactivo.

En primer lugar, se procesa la imagen del mapa (400x400 píxeles, donde el blanco es carretera y el negro es edificio). Para evitar que el coche roce las paredes, se calcula un **subcampo de penalización** alrededor de los obstáculos. 

En segundo lugar, se construye el **campo de propagación**. Para ello, se implementa un algoritmo de onda (Dijkstra) mediante una cola de prioridad (`heapq`). En lugar de sumar la penalización al final (como dice el enunciado de forma literal), se decidió integrar la penalización directamente en el coste de la onda, creando un mapa de costes. Esto evita que el coche se desvíe hacia callejones laterales, tal y como se explicará más adelante.

Por último, se diseña el bucle de navegación. En cada iteración, el coche lee su posición GPS, busca el waypoint (sub-objetivo) bajando por el gradiente del campo y calcula las velocidades V y W para dirigirse hacia él.

## 3. Proceso de desarrollo y evolución

El desarrollo de esta práctica fue iterativo y presentó varios desafíos importantes que obligaron a modificar el enfoque inicial:

**1. El problema de la entrada en callejones laterales:**
En las primeras versiones, se aplicaba la suma literal de los dos subcampos: el campo de propagación (distancia) más el campo de penalización (cercanía a paredes). El problema era que, en los huecos de las paredes (bocacalles), la penalización desaparecía y creaba un "valle" por el que el coche se sentía atraído. El coche se asomaba a la calle lateral, se daba cuenta de que no era el camino y volvía a rectificar. 
**Solución:** Se decidió integrar la penalización dentro del coste de la onda (`nd = d + coste * (1.0 + pen_l[k])`). De esta forma, pasar cerca de una pared encarece el camino, pero un hueco ya no es un valle que atraiga al coche. El camino óptimo sigue recto y por el centro de la calle.

**2. Oscilaciones en las rectas y cortes de esquina:**
El coche oscilaba (hacía zigzag) cuando circulaba por calles rectas, ya que intentaba corregir constantemente su trayectoria. Además, al llegar a las curvas, tendía a cortar la esquina y chocarse contra la pared exterior.
**Solución:** Se implementó un sistema de "dos miradas" (lejana y cercana) y un filtro de suavizado.
*   Se añadió un filtro (`SUAVIZADO`) y una zona muerta (`ZONA_MUERTA`) para evitar que el volante reaccionara a pequeñas variaciones en rectas.
*   Se añadieron dos waypoints: uno lejano (`PASOS_ADELANTE`) para seguir recto con estabilidad, y uno cercano (`PASOS_CURVA`). Si la diferencia de ángulo entre ambos es alta, se detecta una curva, se usa la mirada cercana (para no cortar la esquina) y se frena el coche (`FRENO_CURVA`).

## 4. Funcionamiento del código final

El programa se ejecuta en un bucle a 20 Hz (`Frequency.tick(20)`) y utiliza una máquina de estados (`ESPERANDO`, `NAVEGANDO`, `LLEGADO`).

1.  **Planificación:** Cuando el usuario pica un nuevo destino, el coche se detiene y calcula el campo GPP completo mediante la función `propagar()`. Este campo se dibuja en la interfaz con `WebGUI.showNumpy()` y la ruta óptima se dibuja con `WebGUI.showPath()`.
2.  **Detección de curvas:** En cada iteración, el coche lee su posición. Calcula dos waypoints usando `bajar_gradiente()`: uno a 8 celdas de distancia (lejano) y otro a 4 celdas (cercano). Si la diferencia angular entre ambos supera un umbral (`UMBRAL_CURVA`), el coche entiende que viene una curva.
3.  **Control de velocidades:** 
    *   Si está en recta, usa el ángulo lejano, lo filtra con `SUAVIZADO` e ignora errores menores a `ZONA_MUERTA`. Va a velocidad máxima (`VMAX`).
    *   Si está en curva, usa el ángulo cercano (evitando cortar la esquina), aplica un factor de frenado y reduce la velocidad para girar con seguridad.
    *   La velocidad angular se limita con `WMAX` para no hacer trompos.

## 5. Problemas encontrados

Además del problema de los callejones laterales y las oscilaciones, el principal quebradero de cabeza fue la naturaleza del simulador. Al no tener sensor láser, el coche no puede reaccionar a imprevistos, por lo que el mapa de costes debía estar perfectamente ajustado para que el gradiente lo mantuviera siempre en el centro del carril.

También se probó a implementar un sistema "anti-atasco" que daba marcha atrás cuando el coche chocaba. Sin embargo, esto resultaba muy errático: el coche se salía del mapa o se quedaba atrapado. Se decidió eliminar las marchas atrás y fiarse ciegamente de la planificación global, ajustando la velocidad máxima (`VMAX`) y la ganancia del giro (`KW`) para que el coche tome las curvas con fluidez sin necesidad de retroceder.

## 6. Conclusión

Como conclusión, se ha implementado con éxito un sistema de navegación global basado en el algoritmo GPP. El coche es capaz de leer un mapa estático, trazar una ruta óptima esquivando los edificios y seguirla de forma autónoma hasta el destino seleccionado.

La principal ventaja de este método es que, al tener un mapa global, el coche sabe siempre hacia dónde ir (el plan como recurso), evitando quedarse atrapado en mínimos locales. La integración de la penalización dentro del coste de la onda y el uso de las "dos miradas" han demostrado ser soluciones muy robustas para evitar desviaciones innecesarias y mantener una conducción suave tanto en rectas como en curvas.

## 7. Capturas y vídeo de demostración
