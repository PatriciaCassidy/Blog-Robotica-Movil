# Práctica 1: Aspiradora autónoma (Basic Vacuum Cleaner)

## 1. Introducción
En esta práctica el objetivo es implementar la lógica de un algoritmo de navegación para una aspiradora autónoma. Usando este algoritmo se pretende cubrir el mayor área posible de la casa. Para ello, el robot que se programará dispone de un láser que permite detectar obstáculos y decidir cómo se continua el recorrido. 
La finalidad principal es realizar una exploración pseudoaleatoria, sin utilizar la posición del robot para crear un mapa. El programa que se ha realizado se basa en una máquina de estados con los estados de avanzar, retroceder, girar y espiral.  

## 2. Planteamiento del problema
Lo primero que se hizo fue usar la función parse_laser_data() proporcionada en el enunciado para transformar las 180 medidas del láser en coordenadas polares y cartesianas.  
A continuación, con estas medidas decidí dividir la zona de visión del robot en tres partes: delante, izquierda y derecha.


