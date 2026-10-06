# Práctica 1: Aspiradora autónoma (Basic Vacuum Cleaner)

## 1. Introducción
En esta práctica el objetivo es implementar la lógica de un algoritmo de navegación para una aspiradora autónoma. Usando este algoritmo se pretende cubrir el mayor área posible de la casa. Para ello, el robot que se programará dispone de un láser que permite detectar obstáculos y decidir cómo se continua el recorrido. 
La finalidad principal es realizar una exploración pseudoaleatoria, sin utilizar la posición del robot para crear un mapa. El programa que se ha realizado se basa en una máquina de estados con los estados de avanzar, retroceder, girar y espiral.  

## 2. Planteamiento del problema
Lo primero que se hizo fue usar la función parse_laser_data() proporcionada en el enunciado para transformar las 180 medidas del láser en coordenadas polares y cartesianas.  
A continuación, con estas medidas decidí dividir la zona de visión del robot en tres partes: delante, izquierda y derecha. Para cada una se utiliza la distancia mínima, ya que de esta forma se puede saber si existe algún obstáculo cercano.  
<img width="577" height="303" alt="image" src="https://github.com/user-attachments/assets/82e847f3-e0ef-4d6a-9971-a31d0b189e2b" />  
La lógica principal se encuentra dentro del while True, de manera que el robot está continuamente leyendo el láser y tomando nuevas decisiones. También se utiliza Frequency.tick(50) para controlar la frecuencia del bucle sin utilizar sleep(), taly como se pide en la práctica.

## 3. Máquina de estados
El programa utiliza estos cuatro estados:  
-AVANZAR: el robot se mueve hacia delante mientras no encuentre un obstáculo.  
-RETROCEDER: cuando detecta un obstáculo, retrocede durante un pequeño número de ciclos para separarse de él.  
-GIRAR: después de retroceder, decide si girar hacia la izquierda o hacia la derecha dependiendo de qué lado tenga más espacio. El tamaño del giro se elige de forma aleatoria.  
-ESPIRAL: cuando el robot se encuentra en una zona abierta, realiza una trayectoria circular cuya velocidad lineal aumenta progresivamente, haciendo que la espiral se vaya abriendo.  

La espiral permite recorrer zonas abiertas de una forma diferente al movimiento recto, mientras que los giros aleatorios permiten cambiar de dirección cuando se encuentran paredes u otros obstáculos. Este planteamiento coincide con la idea general de exploración aleatoria y movimiento en espiral propuesta para el ejercicio.

## 4. Proceso de desarrollo
Al principio se probó una solución más sencilla en la que el robot avanzaba y cambiaba de dirección principalmente al encontrar obstáculos. En las primeras pruebas el robot conseguía recorrer algunas zonas, pero acababa quedándose atrapado y repitiendo continuamente una parte del mapa.  

A partir de estas pruebas se modificó la forma de detectar los obstáculos y la manera de realizar los giros. También se probaron diferentes velocidades y diferentes tamaños de giro. Una de las versiones llegó a conseguir aproximadamente un 83% de cobertura, aunque seguía teniendo el problema de repetir demasiado algunas zonas.  

Por ello se decidió no añadir una lógica excesivamente compleja y volver a una solución más sencilla. La versión final mantiene la máquina de estados y utiliza el láser para decidir hacia qué lado girar, añadiendo una pequeña componente aleatoria al ángulo del giro. De esta forma el robot no realiza exactamente el mismo giro cada vez que encuentra una pared.  

## 5. Funcionamiento del código final

En cada iteración se obtienen las medidas del láser y se calculan tres distancias:  

delante = min(laser_data.values[70:111])
izquierda = min(laser_data.values[120:161])
derecha = min(laser_data.values[19:60])

Si la distancia frontal es inferior a 0.45 metros, se considera que existe un obstáculo y se pasa al estado RETROCEDER.

Después de retroceder, se compara el espacio disponible a izquierda y derecha. El robot gira hacia el lado que tiene más espacio y el ángulo del giro se elige aleatoriamente entre 1.2 y 2.5 radianes.

Cuando existe suficiente espacio alrededor del robot, se puede entrar en el estado ESPIRAL. En este estado se mantiene una velocidad angular constante mientras la velocidad lineal aumenta poco a poco, haciendo que el radio de la trayectoria sea cada vez mayor.

El programa no utiliza las coordenadas x e y de la odometría para saber dónde está el robot, solamente utiliza las medidas del láser para tomar decisiones sobre el entorno.  
Fotos del simulador:  
<img width="1917" height="907" alt="image" src="https://github.com/user-attachments/assets/6f2ce4b2-0f37-4ed6-9324-ecd79c8ecdcf" />  

<img width="1908" height="910" alt="image" src="https://github.com/user-attachments/assets/5593316a-6b66-45e8-9fa9-39752710ed7f" />



