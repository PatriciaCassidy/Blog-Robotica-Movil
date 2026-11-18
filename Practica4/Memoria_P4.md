# Práctica 4: Navegación Global con Gradient Path Planning (GPP)

## 1. Introducción

El objetivo de esta práctica es programar un coche autónomo, equipado con un sensor GPS y un mapa de la ciudad, para que ejerza de taxi autónomo. El coche debe ser capaz de planificar su navegación desde el lugar en el que se encuentra hasta otro punto de la ciudad que marque el usuario picando en el mapa, y de conducirse hasta allí por sí mismo.
 
Se trata de un problema de **navegación global**: el destino está lejos, hace falta un mapa y hay que localizarse dentro de él (en este caso mediante el GPS). Para resolverlo se ha utilizado el algoritmo **Gradient Path Planning (GPP)**, que es un ejemplo de *plan como recurso*: en lugar de calcular una secuencia fija de subobjetivos, se construye un campo escalar sobre todo el mapa que el coche puede consultar en cualquier momento para saber hacia dónde moverse para llegar al destino.
 
El campo es la suma de dos componentes:
 
- **Subcampo de propagación a destino:** una onda que se expande desde el destino sin atravesar obstáculos.
- **Subcampo de penalización por cercanía a obstáculos:** sube el valor del campo cerca de las paredes para que el coche se mantenga alejado de ellas.
Además de la planificación, la aplicación pilota el coche siguiendo el gradiente de ese campo, gobernando reactivamente la velocidad lineal (V) y la velocidad angular (W) en cada iteración. Una vez alcanzado el destino, el coche se detiene y espera un nuevo destino.
 
La solución está realizada en Python y utiliza únicamente `HAL`, `WebGUI` y `Frequency` junto con `numpy`, `math` y `heapq`.
 
## 2. Planteamiento del problema
 
### 2.1. El mapa
 
El mapa de la ciudad es una imagen de 400x400 píxeles en escala de grises (`cityLargenBin.png`). El mundo de Gazebo está centrado en [0, 0] y mide 500 m de lado, por lo que cada celda del mapa representa 1,25 m. La carretera es blanca y todo lo demás (edificios, aceras, fuera de calzada) se considera obstáculo.
 
Al arrancar, el programa carga el mapa con `WebGUI.getMap()` y lo convierte en una matriz booleana `libre` (valor > 127 = carretera). El mapa no cambia durante la ejecución, así que todo lo que depende únicamente de él se calcula una sola vez.
 
### 2.2. Conversión de coordenadas
 
El mapa se indexa como `mapa[fila][columna]`, pero las funciones de la interfaz trabajan con coordenadas del mundo y con el par `[columna, fila]`. Para no mezclar sistemas, el código trabaja siempre con celdas `(fila, col)` y encapsula las conversiones en dos funciones:
 
- `mundo_a_celda(x, y)`: usa `WebGUI.worldToGrid()` y devuelve `(fila, col)` limitada a los bordes del mapa.
- `celda_a_mundo(fila, col)`: usa `WebGUI.gridToWorld()` para volver a metros.
Como el enunciado avisa, hay que invertir el orden de las coordenadas al pasar de (i, j) a (j, i), y esto fue una de las fuentes de errores (ver apartado 7).
 
### 2.3. Destino fuera de carretera
 
El usuario puede picar en cualquier punto, incluso encima de un edificio. La función `celda_libre_cercana()` comprueba si la celda es transitable y, si no lo es, busca en radios crecientes (hasta 40 celdas) la celda de carretera más cercana. Ese será el destino real del algoritmo.
 
## 3. Planificación: construcción del campo
 
### 3.1. Subcampo de penalización por cercanía a obstáculos
 
La función `calcular_penalizacion()` se ejecuta una sola vez al inicio. Parte de la máscara de obstáculos y la va dilatando celda a celda (vecindad de 4) durante `R_PEN = 4` iteraciones. En cada iteración, el anillo nuevo que aparece alrededor de la pared recibe una penalización que disminuye con la distancia:
 
`pen = K_PEN · (R_PEN − d + 1)`
 
Con `K_PEN = 1.0` esto da penalizaciones de 4, 3, 2 y 1 para las celdas a 1, 2, 3 y 4 celdas de la pared (entre 1,25 m y 5 m aproximadamente), y 0 para el resto. Es la representación de las ondas reflejadas por las paredes que se mencionan en la teoría: cerca de un obstáculo el campo "sube", y lejos no se nota.
 
### 3.2. Subcampo de propagación a destino
 
La función `propagar(meta)` genera la onda desde el destino mediante un algoritmo de **Dijkstra 8-conectado** con una cola de prioridad (`heapq`):
 
- Se inicializa la celda de destino con valor 0 y todas las demás con infinito.
- Se extrae siempre la celda con menor valor acumulado y se actualizan sus ocho vecinos.
- El coste de un paso es 1,0 en horizontal/vertical y 1,414 en diagonal (aproximación de √2), de modo que las diagonales cuestan más, como sugiere el enunciado.
- No se atraviesan obstáculos, y tampoco se permite **cortar esquinas** en diagonal: un paso diagonal solo es válido si las dos celdas adyacentes que lo flanquean también son libres.
Los dos subcampos se suman dentro del propio coste de cada paso:
 
`nuevo_valor = valor_actual + coste_paso · (1 + penalización_de_la_celda_destino)`
 
De este modo, pasar junto a una pared multiplica el coste del paso (hasta por 5 en la celda pegada a la pared), y el camino de menor coste tiende a ir centrado en la calzada y no pegado a los bordes. El resultado es un campo con forma de embudo cuyo mínimo está en el destino.
 
Para que el bucle de Python sea lo más rápido posible, el mapa y la penalización se aplanan en listas (`ravel().tolist()`) y se trabaja con un índice lineal `fila * ANCHO + col`.
 
**Diferencia respecto al enunciado.** La teoría indica que la propagación se detiene al llegar al punto de partida. En esta solución la onda se propaga por **todo el mapa conectado**. Es un cálculo más costoso, pero se hace una sola vez al fijar cada destino, y a cambio el campo es válido desde cualquier posición del coche. Esto encaja con la idea de plan como recurso: si el coche se desvía, no hace falta replanificar, basta con consultar el campo en la nueva posición.
 
### 3.3. Visualización
 
- `imagen_campo()` convierte el campo en una imagen `uint8` de 400x400 que se muestra con `WebGUI.showNumpy()`. Los valores se normalizan al rango 1–254, de forma que las zonas oscuras están cerca del destino y las claras lejos. Las celdas inalcanzables (obstáculos) se quedan a 0.
- `camino_para_mostrar()` baja por el gradiente desde la posición del coche hasta el destino (con un límite de 4000 pasos) y dibuja el camino resultante con `WebGUI.showPath()`, tomando un punto de cada dos para aligerar el dibujo.
## 4. Pilotaje: bajar por el gradiente
 
### 4.1. Descenso por el gradiente
 
La función `bajar_gradiente(campo, fila, col, pasos)` parte de la celda del coche y en cada paso se mueve al vecino (de los ocho) con menor valor del campo, deteniéndose si ninguno mejora el actual. Esto sirve para obtener puntos "por delante" del coche que son siempre transitables y que siguen el camino planificado.
 
### 4.2. Dos miradas: lejana y cercana
 
Tal como recomienda el enunciado, los puntos objetivo locales se eligen a una distancia suficientemente grande para no pasarse de las curvas y caer en un mínimo local. El problema es que una mirada lejana en una curva "corta" la esquina y una mirada muy cercana hace que el coche oscile en las rectas. Por eso se utilizan **dos miradas**:
 
- **Mirada lejana** (`PASOS_ADELANTE = 8` celdas, unos 10 m): se usa en las rectas, donde da un giro más suave y estable.
- **Mirada cercana** (`PASOS_CURVA = 4` celdas, unos 5 m): se usa cuando hay una curva delante, para seguir el trazado con más fidelidad.
Para cada mirada se calcula el ángulo entre la orientación del coche (`pose.yaw`) y la dirección hacia ese punto, normalizado a [−π, π] con `normaliza()`.
 
### 4.3. Detección de curvas
 
Si el coche va recto, las dos miradas apuntan prácticamente en la misma dirección. Si hay una curva delante, la lejana y la cercana difieren. La diferencia entre ambos ángulos (`curva`) se compara con un umbral (`UMBRAL_CURVA = 0.35 rad`):
 
- **Si hay curva:** el error de ángulo es el de la mirada cercana, y se aplica un factor de freno `factor_curva = 1 − FRENO_CURVA · min(curva / 0.8, 1)`, que reduce más la velocidad cuanto más cerrada sea la curva.
- **Si no hay curva:** el error es el de la mirada lejana pasado por un filtro de suavizado, `error_f = SUAVIZADO · error_f + (1 − SUAVIZADO) · ang_l`, y no se frena.
### 4.4. Cálculo de V y W
 
- Se aplica una **zona muerta** (`ZONA_MUERTA = 0.04 rad`): si el error es menor que ese valor se considera 0, para evitar el zigzag en las rectas.
- **Velocidad angular:** `W = KW · error`, limitada a ±`WMAX`.
- **Velocidad lineal:** `V = VMAX · max(0.3, 1 − |error| / 0.8) · factor_curva`. Es decir, el coche va a toda velocidad cuando el error es pequeño y reduce progresivamente cuanto más tiene que girar, con un suelo del 30 % de `VMAX` para no pararse nunca, y además frena extra si detecta una curva.
El resultado es un controlador reactivo proporcional con tres mecanismos de suavizado (filtro, zona muerta y freno en curva) que se ejecuta a 20 Hz con `Frequency.tick(FREQ)`.
 
## 5. Gestión de destinos: máquina de estados
 
El programa funciona con tres estados:
 
- **ESPERANDO:** el coche está parado y no hay destino válido.
- **NAVEGANDO:** hay campo calculado y el coche lo sigue.
- **LLEGADO:** el coche está a menos de `RADIO_LLEGADA = 4 m` de la celda de destino, se detiene y espera.
En cada iteración se lee `WebGUI.getTargetPose()` y se compara con el último destino atendido (con una tolerancia de 0,01 m). Si es distinto, se considera un **nuevo destino** y se hace lo siguiente: se frena el coche, se reinicia el filtro del error, se ajusta el destino a carretera, se calcula el campo, se muestra en la interfaz y se pasa a NAVEGANDO. Si la celda donde está el coche tiene valor infinito en el campo (es decir, no hay conexión por carretera con el destino), se avisa por consola y se vuelve a ESPERANDO.
 
El destino que ya estuviera marcado al arrancar el programa se ignora (se guarda como `ultimo`), de forma que el coche espera a que el usuario pique en el mapa. Esto permite encadenar tantos destinos como se quiera sin reiniciar la aplicación.
 
## 6. Parámetros de la versión final
 
| Parámetro | Valor | Función |
|---|---|---|
| `R_PEN` | 4 celdas | Radio alrededor de la pared que se penaliza |
| `K_PEN` | 1.0 | Intensidad de la penalización |
| `PASOS_ADELANTE` | 8 celdas | Mirada lejana (rectas) |
| `PASOS_CURVA` | 4 celdas | Mirada cercana (curvas) |
| `UMBRAL_CURVA` | 0.35 rad | Diferencia entre miradas para detectar una curva |
| `FRENO_CURVA` | 0.5 | Cuánto frena en curva |
| `SUAVIZADO` | 0.5 | Filtro del error de ángulo |
| `ZONA_MUERTA` | 0.04 rad | Evita el zigzag en rectas |
| `RADIO_LLEGADA` | 4 m | Distancia al destino para darlo por alcanzado |
| `VMAX` | 20 | Velocidad lineal máxima |
| `KW` | 1.5 | Ganancia del giro |
| `WMAX` | 2.0 | Límite de velocidad angular |
| `FREQ` | 20 Hz | Frecuencia del bucle |
 
## 7. Problemas encontrados  
 
- **Orden de coordenadas fila/columna.** La interfaz devuelve `[columna, fila]` mientras que el mapa se indexa `[fila][columna]`. Un error en este orden hace que el coche aparezca "fuera de carretera" y que el campo se calcule en un sitio distinto al esperado. Por eso se encapsularon las conversiones en `mundo_a_celda` y `celda_a_mundo`, y se añadió una comprobación al inicio que avisa por consola si el coche sale fuera de la carretera.
- **Destino fuera de carretera.** Si el usuario pica sobre un edificio, no hay camino posible hacia esa celda. Se resolvió buscando la celda de carretera más cercana.
- **Destino inalcanzable.** Algunas zonas de carretera pueden no estar conectadas con la posición del coche. Al propagar por todo el mapa, esas celdas quedan con valor infinito y se detecta fácilmente, evitando que el coche se quede dando vueltas.
- **Coche pegado a las paredes.** Con solo la distancia, el camino más corto roza las esquinas. Se añadió el subcampo de penalización y se prohibió cortar esquinas en diagonal en la propagación.
- **Zigzag en las rectas.** Un control proporcional puro oscilaba alrededor de la dirección correcta. Se solucionó con una zona muerta y con un filtro de suavizado del error.
- **Curvas cortadas o pasadas.** Con una única mirada, si era corta el coche oscilaba, y si era lejana cortaba las curvas y se acercaba a los edificios. Se solucionó con las dos miradas y la detección de curva por la diferencia entre ambas.
- **Velocidad en curva.** Ir siempre a `VMAX` provocaba salidas de la carretera en las curvas cerradas, por lo que se añadió el freno proporcional a lo cerrada que es la curva, y un suelo del 30 % para no detenerse.
- **Coste de la propagación.** Recorrer 160.000 celdas con Dijkstra en Python puro es lento. Se aplanaron las estructuras a listas y se calcula una sola vez por destino, mientras que la penalización se calcula una sola vez para todo el programa.
## 8. Funcionamiento del código final
 
En cada iteración del bucle principal se realizan los siguientes pasos:
 
1. Se leen la posición del coche (`HAL.getPose3d()`) y el destino marcado en la interfaz (`WebGUI.getTargetPose()`).
2. Si el destino es nuevo, se frena el coche, se ajusta el destino a carretera, se calcula el campo con `propagar()`, se muestra el campo y el camino, y se pasa a NAVEGANDO (o se avisa si es inalcanzable).
3. Si el estado es NAVEGANDO, se comprueba si el coche está a menos de 4 m del destino. Si es así, se para y pasa a LLEGADO.
4. Si no ha llegado, se calculan las dos miradas bajando por el gradiente (8 y 4 celdas) y sus ángulos respecto a la orientación del coche.
5. Se decide si hay curva comparando los dos ángulos, y se elige el error y el factor de freno correspondientes.
6. Se aplica la zona muerta y se calculan W (proporcional y limitada) y V (en función del error y de la curva).
7. Se envían las órdenes con `HAL.setV()` y `HAL.setW()`. En los estados ESPERANDO y LLEGADO se mantiene el coche parado.
8. Se regula la frecuencia del bucle con `Frequency.tick(FREQ)`.  
## 9. Conclusión
 
Se ha desarrollado un taxi autónomo que planifica su ruta sobre el mapa de la ciudad mediante Gradient Path Planning y la recorre pilotando de forma reactiva.
 
La planificación genera un campo escalar que suma dos subcampos: una propagación tipo Dijkstra desde el destino, que garantiza el camino más corto sin atravesar obstáculos, y una penalización por cercanía a las paredes, que mantiene el coche centrado en la calzada. Al calcularse sobre todo el mapa, el campo se comporta como un plan como recurso: sirve desde cualquier posición y el coche solo tiene que consultarlo. El pilotaje baja por el gradiente utilizando dos puntos de mirada, uno lejano para las rectas y otro cercano para las curvas, y traduce el error de ángulo en velocidad angular y lineal, con zona muerta, suavizado y freno en curva. Una máquina de estados permite que el coche espere y atienda nuevos destinos de forma indefinida.
 
La principal ventaja del método es que la planificación es completa y el comportamiento resultante es sencillo y robusto, sin los mínimos locales de los campos de potencial puros, porque el campo se construye con una onda desde el destino. Como contrapartida, requiere un mapa conocido y estático, no tiene en cuenta obstáculos dinámicos (otros coches, peatones) y el coste de calcular el campo crece con el tamaño del mapa. Una posible mejora sería combinarlo con una capa de navegación local (como la VFF de la práctica anterior) para esquivar obstáculos imprevistos, siguiendo la arquitectura híbrida de dos capas vista en teoría.
 
## 10. Capturas y vídeo de demostración
 
### Capturas del simulador  
 <img width="1324" height="1336" alt="image" src="https://github.com/user-attachments/assets/c77ffa84-4c99-4f94-b798-b070091732b5" />
 
### Vídeo de demostración   
https://github.com/user-attachments/assets/1169e535-3515-46de-bf4d-bc3075eefb6e




