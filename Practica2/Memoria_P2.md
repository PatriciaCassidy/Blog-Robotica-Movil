

# Práctica 2: Seguimiento de línea (Follow Line)

## 1. Introducción

En esta práctica el objetivo es programar un coche autónomo capaz de seguir una línea roja situada en el centro de un circuito. Para ello, el coche utiliza una cámara frontal que permite detectar la línea y decidir cómo debe modificar su movimiento.

El objetivo principal es completar los circuitos en el menor tiempo posible, intentando mantenerse cerca de la línea y evitando oscilaciones excesivas, especialmente en las curvas. Además, el comportamiento debe ser robusto para que el coche pueda continuar cuando la línea desaparece temporalmente de la imagen.

La solución realizada en Python se basa en el procesamiento de imágenes y en un controlador PID para calcular el giro del volante. También se adapta la velocidad del coche dependiendo de la trayectoria detectada.

## 2. Planteamiento del problema

Lo primero que se hace es obtener la imagen de la cámara del coche y procesarla utilizando OpenCV. Para detectar la línea roja, se transforma la imagen del espacio de color BGR a HSV. Este espacio permite seleccionar los tonos rojos mediante dos rangos, uno para cada extremo del color rojo.

Una vez obtenida la máscara, se divide la imagen en dos franjas horizontales:

- **Zona cercana:** se utiliza para conocer la posición de la línea en la parte inferior de la imagen, que es la más próxima al coche.
- **Zona lejana:** permite estimar hacia dónde se dirige la línea antes de que el coche llegue a esa parte del circuito.

En cada zona se buscan los fragmentos rojos mediante componentes conectados. De los fragmentos detectados se selecciona el que está más cerca de la posición de referencia, evitando elegir partes de la imagen que no correspondan a la línea que se estaba siguiendo.

A partir de las posiciones detectadas se calculan dos valores: el rumbo, que indica hacia qué lado se dirige la línea, y el desplazamiento, que representa la diferencia entre las posiciones de la zona cercana y la lejana.

## 3. Control del coche

El programa funciona mediante un bucle principal que obtiene imágenes, procesa la información y calcula las órdenes de velocidad lineal y angular en cada iteración.

### 3.1. Control del volante mediante PID

Para calcular el giro se combinan varios términos de control:

- **Proporcional del rumbo:** permite orientar el coche siguiendo la dirección que marca la línea.
- **Corrección adicional en curvas:** aumenta el giro cuando el rumbo detectado es elevado, ayudando a afrontar curvas más cerradas.
- **Proporcional del desplazamiento:** corrige la diferencia entre la posición cercana y la lejana de la línea.
- **Derivativo:** tiene en cuenta los cambios del rumbo entre imágenes y ayuda a amortiguar las correcciones bruscas.
- **Integral:** acumula el desplazamiento para corregir desviaciones persistentes.

La salida del controlador determina la velocidad angular del coche. Para evitar giros demasiado bruscos, se limita su valor máximo. También se filtra la derivada y se limita la integral para evitar que se acumule en exceso.

### 3.2. Ajuste de la velocidad

La velocidad lineal se adapta a la trayectoria detectada. Cuando el coche está en una recta puede alcanzar una velocidad mayor, mientras que cuando detecta una curva reduce la velocidad para facilitar el seguimiento de la línea.

También se tiene en cuenta la magnitud del rumbo y del giro calculado. De esta manera, el coche no depende únicamente de si está en una recta o en una curva, sino también de cuánto necesita girar.

La aceleración y el frenado se realizan progresivamente mediante incrementos y decrementos de velocidad. Además, el coche solo aumenta la velocidad cuando el desplazamiento está suficientemente controlado, para evitar acelerar mientras todavía está corrigiendo su posición.

## 4. Recuperación cuando se pierde la línea

Uno de los objetivos de la práctica es evitar que el coche se quede sin control cuando la línea desaparece temporalmente de la cámara.

Para ello, el programa contempla varias situaciones:

- **SIGUE:** se detecta la línea y el coche continúa utilizando el controlador para seguirla.
- **CURVA:** la línea cercana se detecta, pero no aparece correctamente en la zona lejana. Se interpreta que puede estar comenzando una curva y se reduce la velocidad.
- **HUECO:** si la línea deja de detectarse durante unas pocas imágenes, el coche frena y mantiene temporalmente el giro hacia el último lado conocido.
- **BUSCA:** si la pérdida continúa, se intenta localizar de nuevo el rojo en una zona más amplia de la imagen.
- **PERDIDA:** si no se encuentra ningún fragmento rojo, el coche avanza lentamente mientras gira para intentar recuperar la línea. Si tarda demasiado, cambia el sentido del giro.

Cuando vuelve a detectar la línea, se reinician las variables necesarias del controlador para evitar que los valores anteriores provoquen una corrección brusca.

## 5. Proceso de desarrollo

Durante el desarrollo se ha buscado un equilibrio entre la velocidad del coche y la estabilidad del seguimiento. Aumentar demasiado la velocidad puede provocar que el coche se salga de la línea en las curvas, mientras que reducirla en exceso hace que tarde más en completar el circuito.

Por este motivo, se han incluido diferentes parámetros para regular la velocidad máxima, la velocidad mínima, la aceleración, el frenado y las ganancias del controlador.

También se ha prestado atención a la detección de la línea. En lugar de elegir cualquier fragmento rojo de la imagen, se utiliza la posición anterior como referencia para intentar mantener un seguimiento continuo del mismo fragmento.

La solución final combina el seguimiento visual, el control PID, la adaptación de la velocidad y varios mecanismos de recuperación. Los parámetros se mantienen iguales para los diferentes circuitos, con el objetivo de conseguir un comportamiento lo más general posible.

## 6. Funcionamiento del código final

En cada iteración del bucle principal se realizan los siguientes pasos:

1. Se obtiene una nueva imagen de la cámara.
2. Se detectan los píxeles rojos mediante una máscara HSV.
3. Se divide la imagen en una zona cercana y otra lejana.
4. Se buscan los fragmentos rojos y se seleccionan según su posición y distancia respecto a las referencias.
5. Se calculan el rumbo y el desplazamiento de la línea.
6. Se determina si el coche está siguiendo la línea, entrando en una curva o intentando recuperarla.
7. Se calcula el giro mediante los términos proporcional, derivativo e integral.
8. Se ajusta la velocidad lineal según la trayectoria y la corrección necesaria.
9. Se envían las órdenes de movimiento al coche y, periódicamente, se muestra una imagen de depuración.

Además, el programa comprueba si la imagen ha cambiado antes de volver a procesarla y muestra información de diagnóstico, como el estado del coche, el rumbo, el desplazamiento, la velocidad y el giro. Esto facilita analizar el comportamiento durante las pruebas.

## 7. Problemas encontrados

Durante el desarrollo de la práctica aparecieron varios problemas relacionados con el seguimiento de la línea roja. Uno de los más importantes fueron las oscilaciones del coche, especialmente al salir de las curvas y volver a las rectas, además de la falta de giro en algunas curvas cerradas. También hubo dificultades para detectar correctamente la línea en ciertos tramos y recuperarla cuando se perdía. Para mejorar el comportamiento, se ajustaron los parámetros del controlador y se utilizaron diferentes zonas de la imagen para calcular la dirección del coche. Otro problema fue el tiempo de recorrido, que superaba el objetivo inicial, junto con la baja frecuencia de actualización de la cámara, que dificultaba un control estable. Además, fue necesario corregir algunos detalles de los imports y comprobar el sentido de giro del vehículo. Finalmente, queda probar la última versión en todos los circuitos y realizar los ajustes necesarios para mejorar la estabilidad y reducir el tiempo de recorrido.

## 8. Conclusión

Como conclusión, se ha desarrollado un controlador visual para un coche autónomo que utiliza una cámara frontal para detectar una línea roja y seguir el recorrido de un circuito.

El programa combina el procesamiento de imágenes con un controlador PID para calcular el giro y adapta la velocidad según la trayectoria detectada. Además, incluye mecanismos para continuar durante pequeñas pérdidas de la línea e intentar recuperarla cuando desaparece durante más tiempo.

El objetivo final es completar distintos circuitos de forma estable y en el menor tiempo posible, manteniendo el coche cerca de la línea y reduciendo las oscilaciones. El comportamiento definitivo depende de los parámetros utilizados y de los resultados obtenidos durante las pruebas en el simulador.

## 9. Capturas y vídeo de demostración

### Capturas del simulador


### Vídeo de demostración
