# Práctica 3: Navegación local con campos de fuerzas virtuales (VFF)
 
## 1. Introducción  
El objetivo de esta práctica es programar la navegación local de un coche de Fórmula 1 que debe recorrer un circuito con vallas en los laterales y con otros coches repartidos por el recorrido como obstáculos. El coche dispone de un sensor láser y de un sensor GPS, y no se utiliza la cámara ni ningún mapa del entorno.  

El sistema de navegación global ya ha planificado una secuencia de subobjetivos (waypoints) que, si se van alcanzando de forma consecutiva, garantiza que se completa la vuelta. El trabajo de esta práctica consiste en la navegación local: ir hacia el siguiente subobjetivo esquivando los obstáculos que aparezcan por el camino.  

La solución realizada en Python se basa en el algoritmo VFF. Los obstáculos generan fuerzas repulsivas, el subobjetivo genera una fuerza atractiva, y la combinación de ambas se traduce en las dos órdenes que recibe el coche: la velocidad lineal (V) y la velocidad angular (W). El objetivo es completar el circuito en el menor tiempo posible sin chocar con ningún obstáculo.

## 2. Planteamiento del problema
El algoritmo se ejecuta en un bucle infinito. En cada iteración se leen la posición del coche y las 180 medidas del láser, y se calculan dos vectores expresados en el sistema de referencia del coche, con el eje X hacia delante y el eje Y hacia la izquierda. Este sistema es el más cómodo para calcular y combinar las fuerzas, ya que permite saber directamente hacia qué lado debe girar el coche.
 
- **Fuerza atractiva:** apunta hacia el waypoint actual. Las coordenadas del destino vienen en el sistema absoluto, por lo que se calcula el vector que va del coche al destino y se rota con el ángulo de orientación (yaw) del coche para pasarlo a su sistema de referencia. Después se normaliza para que tenga módulo 1, de forma que la atracción sea constante independientemente de la distancia.
  
- **Fuerza repulsiva:** cada rayo del láser que detecta algo dentro de su rango genera un vector en sentido contrario al rayo, cuyo módulo es inversamente proporcional a la distancia medida (`1/d`). Para que no haya saltos, a este valor se le resta `1/límite`, de forma que la fuerza vale cero justo en el límite del rango. La fuerza repulsiva total es la suma de las contribuciones de todos los rayos.
El rango de los rayos no es el mismo para todos. Los rayos frontales (aproximadamente ±30°) ven hasta 5 metros, para detectar a los otros coches con antelación. Los rayos laterales solo ven hasta 2 metros, para que el coche no reaccione de forma exagerada a las vallas.   

<img width="846" height="1092" alt="image" src="https://github.com/user-attachments/assets/f569ac84-7e4d-4632-a692-f8f36e81b56b" />  

<img width="730" height="1038" alt="image" src="https://github.com/user-attachments/assets/dc062d67-c54b-4f65-8fc6-a8f9c5dd0474" />  

## 3. Traducción de la fuerza a velocidades
La fuerza resultante se combina y después se traduce en las órdenes V y W.
 
### 3.1. Combinación de las fuerzas
 
El enunciado indica que la fuerza repulsiva suele apuntar al revés de la marcha, por lo que no se puede sumar directamente a la atractiva sin más. Como el coche debe seguir avanzando, la componente hacia delante se mantiene siempre positiva y es la componente lateral la que gobierna el giro:  
 
- `fx = max(ALPHA · ax, 0.1)`: componente hacia delante, que solo depende de la atracción y nunca baja de 0.1.  
- `fy = ALPHA · ay + BETA · ry`: componente lateral, que suma la atracción y la repulsión.  
Los pesos `ALPHA` y `BETA` regulan el carácter del coche: un `BETA` alto lo hace más prudente con los obstáculos, mientras que un `ALPHA` alto lo hace ir más directo al destino. Además, la repulsión lateral se multiplica por un factor empírico de 0.8 para suavizar el giro.  
 
### 3.2. Velocidad angular (W)  
 
A partir de la fuerza resultante se calcula su ángulo respecto al eje de avance, `error = atan2(fy, fx)`. Si la fuerza apunta recto hacia delante el ángulo es 0 y el coche no gira. Cuanto más se desvía la fuerza hacia un lado, mayor es el giro. La velocidad angular es proporcional a ese ángulo, `w = KP · error`.  
 
### 3.3. Velocidad lineal (V)  
 
La velocidad lineal sale de dos límites y se queda con el más restrictivo:  
 
- **Límite por giro:** el coche va a la velocidad máxima cuando la fuerza apunta recto, y reduce la velocidad cuanto más tiene que girar. Esta reducción tiene un suelo del 50 % de la velocidad máxima.  
- **Límite por obstáculo frontal:** se mide la distancia mínima en el cono central del láser. Si hay algo delante, el coche frena de forma progresiva a medida que se acerca. Si no hay nada a menos de unos 6 metros, no frena. Esta frenada tiene un suelo del 25 % de la velocidad máxima, de modo que el coche nunca se detiene al esquivar.  

## 4. Proceso de desarrollo
Un subobjetivo se considera alcanzado cuando el coche está a menos de 2 metros de su posición. En ese momento se marca como alcanzado y se pasa a navegar hacia el siguiente. Usar un umbral de 2 metros en lugar de uno más pequeño evita que el coche intente pasar justo por encima de cada punto y le permite suavizar las curvas.  

## 5. Funcionamiento del código final  
Durante el desarrollo se ha buscado un equilibrio entre la velocidad del coche y la seguridad al esquivar. Aumentar la velocidad hace que el circuito se complete antes, pero deja menos tiempo y espacio para reaccionar ante un coche, mientras que ir despacio hace que el tiempo de vuelta sea demasiado alto.  
 
La primera versión sumaba directamente las dos fuerzas completas, pero daba problemas de giros bruscos, por lo que se pasó a mantener la componente frontal siempre positiva. A partir de ahí se fueron ajustando los parámetros de forma experimental, probando una vuelta tras cada cambio para saber qué modificación mejoraba o empeoraba el comportamiento. Se ajustaron el peso de la repulsión, los rangos de visión del láser, la distancia de frenada, la velocidad máxima y la ganancia del giro.  
 
También se probaron otras versiones más rápidas, con más velocidad máxima y con velocidades mínimas distintas. Finalmente se decidió mantener la versión inicial, porque era la que ofrecía un comportamiento más fiable al esquivar los obstáculos.  
 
Parámetros de la versión final: `ALPHA = 1.0`, `BETA = 0.6`, `D_MAX = 2.0 m` (rayos laterales), `D_FRENTE = 5.0 m` (rayos frontales), `FRENO = 5.0`, `VMAX = 11.0`, `KP = 1.2`.  

## 6. Problemas encontrados  
En cada iteración del bucle principal se realizan los siguientes pasos:  
 
1. Se obtienen la posición del coche y las medidas del láser.  
2. Se comprueba si se ha alcanzado el waypoint actual. Si es así, se marca como alcanzado y se pasa al siguiente.  
3. Se calcula la fuerza atractiva: vector unitario hacia el waypoint, en el sistema de referencia del coche.  
4. Se calcula la fuerza repulsiva sumando la contribución de cada rayo del láser que detecta algo dentro de su rango.  
5. Se combinan ambas fuerzas: la componente hacia delante se mantiene siempre positiva y la repulsión actúa en la componente lateral.
6. Se dibujan las fuerzas en la interfaz con `WebGUI.showForces()` para poder depurar el comportamiento.  
7. Se calcula el ángulo de la fuerza resultante y con él la velocidad angular.
8. Se calcula la velocidad lineal según el giro necesario y la distancia al obstáculo frontal.  
9. Se envían las órdenes con `HAL.setV()` y `HAL.setW()` y se controla la frecuencia del bucle con `Frequency.tick()`.  

## 7. Conclusión
Durante el desarrollo aparecieron varios problemas relacionados principalmente con el equilibrio entre velocidad y seguridad. Al principio, la suma de las fuerzas completas provocaba giros bruscos y oscilaciones, por lo que se decidió mantener positiva la componente hacia delante y utilizar la repulsión únicamente para modificar el giro lateral. También se comprobó que un valor pequeño de `BETA` hacía que el coche apenas esquivase los obstáculos, por lo que se aumentó hasta 0.6. Otro problema fueron los choques al aumentar la velocidad, ya que el coche no tenía tiempo suficiente para desviarse de otros vehículos, así que se recuperaron velocidades más bajas durante los giros y las maniobras de esquiva. Además, se ajustaron los rangos del láser, utilizando 2 m para los laterales y 5 m para la zona frontal, y se incorporó una frenada progresiva según la distancia al obstáculo. Durante las pruebas también aparecieron errores de indentación que provocaban el mensaje "Failed to run the application", y se descartaron algunas versiones con una velocidad mínima explícita porque no ofrecían el comportamiento esperado. Por otro lado, la lentitud del simulador (RTF entre 0.06 y 0.18) dificultaba valorar los cambios, ya que el tiempo real no coincidía con el tiempo simulado. Finalmente, hay que tener en cuenta que el método VFF depende únicamente de las medidas actuales del láser, por lo que puede presentar mínimos locales y oscilaciones en espacios estrechos. Tras realizar las distintas pruebas, se decidió conservar la versión que ofrecía el comportamiento más adecuado, priorizando evitar colisiones frente a conseguir una mayor velocidad.  

## 8. Capturas y vídeo de demostración
 Como conclusión, se ha desarrollado un controlador de navegación local basado en campos de fuerzas virtuales que permite al coche recorrer el circuito siguiendo los waypoints y esquivando a los otros coches utilizando únicamente el láser.  
 
El programa calcula en cada iteración una fuerza atractiva hacia el destino y una fuerza repulsiva generada por los obstáculos, las combina manteniendo siempre el avance hacia delante, y traduce el resultado en una velocidad angular proporcional al ángulo de la fuerza y una velocidad lineal que se adapta al giro y a la distancia al obstáculo frontal.  
 
La principal ventaja del método es su sencillez y su rapidez de cálculo, que le permite reaccionar bien ante imprevistos. Como contrapartida, al no usar un mapa solo ve lo que detecta el láser en cada momento, y el compromiso entre velocidad y seguridad depende mucho de los parámetros elegidos, que se han ajustado de forma experimental en el simulador.  

### Capturas del simulador
 
[Añadir capturas]
 
### Vídeo de demostración
 
[Añadir vídeo]
