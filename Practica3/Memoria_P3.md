# Práctica 3: Navegación local con campos de fuerzas virtuales (VFF)
 
## 1. Introducción  
El objetivo de esta práctica es programar la navegación local de un coche de Fórmula 1 que debe recorrer un circuito con vallas en los laterales y con otros coches repartidos por el recorrido como obstáculos. El coche dispone de un sensor láser y de un sensor GPS, y no se utiliza la cámara ni ningún mapa del entorno.  

El sistema de navegación global ya ha planificado una secuencia de subobjetivos (waypoints) que, si se van alcanzando de forma consecutiva, garantiza que se completa la vuelta. El trabajo de esta práctica consiste en la navegación local: ir hacia el siguiente subobjetivo esquivando los obstáculos que aparezcan por el camino.  

La solución realizada en Python se basa en el algoritmo VFF. Los obstáculos generan fuerzas repulsivas, el subobjetivo genera una fuerza atractiva, y la combinación de ambas se traduce en las dos órdenes que recibe el coche: la velocidad lineal (V) y la velocidad angular (W). El objetivo es completar el circuito en el menor tiempo posible sin chocar con ningún obstáculo.

## 2. Planteamiento del problema
El algoritmo se ejecuta en un bucle infinito. En cada iteración se leen la posición del coche y las 180 medidas del láser, y se calculan dos vectores expresados en el sistema de referencia del coche, con el eje X hacia delante y el eje Y hacia la izquierda. Este sistema es el más cómodo para calcular y combinar las fuerzas, ya que permite saber directamente hacia qué lado debe girar el coche.  

Fuerza atractiva: apunta hacia el waypoint actual. Las coordenadas del destino vienen en el sistema absoluto, por lo que se calcula el vector que va del coche al destino y se rota con el ángulo de orientación (yaw) del coche para pasarlo a su sistema de referencia. Después se normaliza para que tenga módulo 1, de forma que la atracción sea constante independientemente de la distancia.  
Fuerza repulsiva: cada rayo del láser que detecta algo dentro de su rango genera un vector en sentido contrario al rayo, cuyo módulo es inversamente proporcional a la distancia medida (1/d). Para que no haya saltos, a este valor se le resta 1/límite, de forma que la fuerza vale cero justo en el límite del rango. La fuerza repulsiva total es la suma de las contribuciones de todos los rayos.  

El rango de los rayos no es el mismo para todos. Los rayos frontales (aproximadamente ±30°) ven hasta 5 metros, para detectar a los otros coches con antelación. Los rayos laterales solo ven hasta 2 metros, para que el coche no reaccione de forma exagerada a las vallas.  

## 3. Traducción de la fuerza a velocidades


## 4. Proceso de desarrollo


## 5. Funcionamiento del código final


## 6. Problemas encontrados


## 7. Conclusión


## 8. Capturas y vídeo de demostración
 
### Capturas del simulador
 
[Añadir capturas]
 
### Vídeo de demostración
 
[Añadir vídeo]
