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




