---
title: "Efecto De Fuego Con ClojureScript"
date: 2019-05-15T19:08:01-07:00
draft: false
---

_This article was [originally posted in the devz.mx blog](https://devz.mx/efecto-de-fuego-con-clojurescript/)._

Terminé de leer el libro _Game Engine Black Book: Doom_ de [Fabien Sanglard](http://fabiensanglard.net/gebbdoom/index.html) recientemente, y uno de los capítulos que más disfruté fue el dedicado a los _ports_ de Doom a las diferentes consolas de la época. Está lleno de detalles técnicos e historias sobre las restricciones sobre las cuales tenían que trabajar los programadores de la época para llevar uno de los títulos más populares y emblemáticos de la PC a consolas "imposibles" como el SNES.

Sin duda alguna uno de los ports más populares fue el de _PlayStation_. Igual que otras consolas de la época, no contaba con suficiente _oomph_ para utilizar los mapas originales, así que usó los mapas modificados para _Jaguar_ que reducen la geometría y el uso de texturas, se redujo el tamaño de las texturas y los frames de animación de algunos de los enemigos. Incluso se dejo al _Archvile_ completamente fuera ya que cuenta con demasiados frames de animación y por lo tanto demasiadas texturas como para caber junto con todo el resto de los datos en 3 MB de memoria RAM.

Pero no todo fueron malas noticias. De hecho el _port_ mejoró algunas cosas, entre ellas los efectos de sonido, la música fue mejorada a 44KHz, 16-bit stereo, o sea calidad de CD (obviamente, considerando que CD era uno de los puntos principales de _PlayStation_). Y lo más significativo fue el uso de _alpha blending_ en sectores todas las texturas de ciertos sectores y puertas, agregando un efecto de color a aquellas puertas que se abren con una tarjeta roja, por ejemplo.

Pero una de las cosas que más llamó mi atención fue los efectos de fuego en las áreas abiertas de ciertos niveles, y en el intro.

![Doom Fire](/images/doom_fire.jpg)

La razón por la que me llamó mi atención es porque a pesar de tener las limitantes del _hardware_, al parecer a los programadores les quedaron algunos ciclos del CPU y no dejaron pasar la oportunidad de hacer algo atractivo con eso. Bien por ellos.

Afortunadamente el autor del libro [cubre precisamente este efecto](http://fabiensanglard.net/doom_fire_psx/index.html) en un artículo en su sitio web. O pueden quedarse y hacer el suyo.

## ¿Como hacemos fuego?

Este es un efecto clásico. Se basa en dos conceptos muy simples:

1. Una paleta de colores para asemejar los colores que encontramos en un fuego real.
2. Propagar el fuego de abajo hacia arriba, calculando el "calor" de pixeles superiores basados en el calor de pixeles actuales.

## Paleta de colores

La paleta de colores consta de colores obscuros, pasando por tonos rojos, amarillos hasta llegar al completamente blanco:

{{< firepalette >}}

Este último color blanco va a servir de "combustible". Y justo como un fuego real, los píxeles superiores van a recibir "calor" de las lineas inferiores. Entonces lo primero que hacemos es organizar nuestra paleta de colores:

```clojure
[
 [  7   7   7] ;; 0
 [ 31   7   7] ;; 1
 [ 47  15   7] ;; 2
 [ 71  15   7] ;; 3
 [ 87  23   7] ;; 4
 [103  31   7] ;; 5
 [119  31   7] ;; 6
 [143  39   7] ;; 7
 [159  47   7] ;; 8
 [175  63   7] ;; 9
 [191  71   7] ;; 10
 [199  71   7] ;; 11
 [223  79   7] ;; 12
 [223  87   7] ;; 13
 [223  87   7] ;; 14
 [215  95   7] ;; 15
 [215 103  15] ;; 16
 [207 111  15] ;; 17
 [207 119  15] ;; 18
 [207 127  15] ;; 19
 [207 135  23] ;; 20
 [199 135  23] ;; 21
 [199 143  23] ;; 22
 [199 151  31] ;; 23
 [191 159  31] ;; 24
 [191 159  31] ;; 25
 [191 167  39] ;; 26
 [191 167  39] ;; 27
 [191 175  47] ;; 28
 [183 175  47] ;; 29
 [183 183  47] ;; 30
 [183 183  55] ;; 31
 [207 207 111] ;; 32
 [223 223 159] ;; 33
 [239 239 199] ;; 34
 [255 255 255] ;; 35
]
```

Es un vector de vectores. Cada vector representa un color en RGB, y la posición en el vector que los contiene va a servir de índice, que va de más "frio" (0) a más "caliente" (35).

## Encendiendo el fuego

Para encender el fuego necesitamos combustible, y el estado del fuego lo podemos representar como un arreglo de píxeles. El número de píxeles es igual al número total de píxeles en nuestra ventana. Por ejemplo si la ventana es de `160x84` entonces el número total de píxeles es de 13,440. A este arreglo de píxeles le estoy llamando `fire-pixels`.

Dado a que el arreglo de píxeles es lineal, cada 160 píxeles representa un renglón de nuestro fuego. Entonces necesitamos llenar de 0 todas las posiciones, excepto las que representen el último renglón:

```clojure
(-> state
    (update-in [:fire-pixels]
               #(reduce (fn [px i]
                          (conj px (if (< i (- pixel-count pixel-row)) 0 35)))
                        [] (range pixel-count))))
```

`state` es un mapa que contiene el estado de nuestro programa. El `reduce` anterior es un ciclo que va de `0 <= i < pixel-count` e inicializa nuestro fuego.

## Propagando el fuego

Esta es la parte interesante, y en realidad es bastante sencillo. Por cada `tick`, vamos a recalcular todos los píxeles en el arreglo `fire-pixels`. Utilizamos dos funciones para lograrlo:

```clojure
(defn- do-fire
  [{:keys [fire-pixels pixel-count] :as state}]
  (-> state
      (update-in [:fire-pixels]
                 #(loop [i 0 px fire-pixels]
                    (if (< i pixel-count)
                      (let [[random-index pixel] (spread-fire-random state i)
                            random-index (Math/abs (- i random-index))]
                        (recur (inc i) (assoc px random-index pixel)))
                      px)))))
```

Es muy similar al `reduce` anterior. Simplemente es un ciclo que va de `0 <= i < pixel-count` y en base al valor actual de cada píxel, calcula un nuevo valor. Después asigna ese nuevo valor a una posición aleatoria en el arreglo.

Esto logra dos objetivos:

1. Simular movimiento en el fuego, gracias a que el arreglo es constantemente recalculado.
2. Propagar el "calor" del fuego de abajo hacia arriba.

Finalmente la función `spread-fire-random` que es donde se hace el cálculo del valor del píxel de acuerdo a su valor actual:

```clojure
(defn- spread-fire-random
  [{:keys [fire-pixels pixel-row] :as state} src]
  (let [pixel (get fire-pixels (+ src pixel-row))
        random-index (bit-and (Math/round (* (Math/random) 3.0)) 3)]
    (cond
      (= pixel 0) [random-index 0]
      (nil? pixel) [random-index (get fire-pixels src)]
      :else [(bit-and random-index 1) (- pixel (bit-and random-index 1))])))
```

Esta funcion realiza lo siguiente:

1. Para un píxel `n` obtiene el valor actual del píxel que se encuentra debajo de el `(get fire-pixels (+ src pixel-row))` recordando que la esquina superior izquierda representa las coordenadas 0,0. A este valor le llamamos `pixel`.
2. Calcula un índice aleatorio.
3. Si `pixel = 0` entonces regresa la tupla `[random-index 0]`. El segundo elemento representa el nivel de calor del nuevo píxel. En este caso como el píxel debajo tiene nivel 0, eso quiere decir que ya llegó a su nivel mínimo y por lo tanto debe ser 0.
4. Si `pixel = nil` quiere decir que estamos revisando más allá de los límites de la ventana, por lo que regresamos el valor del píxel actual `[random-index (get fire-pixels src)]`
5. Si píxel no es 0 y tampoco es `nil`, entonces el nivel de calor es la diferencia entre el nivel actual y un valor aleatorio: `[(bit-and random-index 1) (- pixel (bit-and random-index 1))]`.

Cuando `do-fire` recibe la tupla, los separa en `random-index` y `pixel` provenientes de la posición 0 y 1 respectivamente. Y de ahí simplemente modifica el arreglo `fire-pixels` en la posición `random-index` y le asigna el valor de `pixel`.

Lo único que queda es pintar la pantalla.

## Pintando fuego

Esto no tiene nada que ver con la simulación, pero igual lo incluyo por si le sirve de referencia a alguien.

La función `draw-pixels` recibe el arreglo `fire-pixels` y la paleta de colores, y modifica los píxeles en pantalla.

```clojure
(defn- draw-pixels
  [{:keys [fire-pixels pallete] :as state}]
  (let [px (q/pixels)  ;; screen pixels
        px-count (* 4 (* (* (q/display-density) (q/width))
                         (* (q/display-density) (q/height))))
        px-row (* 4 (* (q/display-density) (q/width)))
        [width height] screen-dimension]
    (loop [i 0]
      (when (< i px-count)
        (let [pixel-value (get fire-pixels (/ i 4))
              [r g b] (get pallete pixel-value)]
          (aset px (+ i 0) r)
          (aset px (+ i 1) g)
          (aset px (+ i 2) b)
          (aset px (+ i 3) 255))
        (recur (+ i 4))))
    (q/update-pixels)))
```

`(q/pixels)` regresa los píxeles de la pantalla. La librería que estamos utilizando (Quil) representa los píxeles como un conjunto de 4 números enteros representando el componente rojo, verde, azul y el nivel de alpha. Entonces por cada `fire-pixel` vamos a tener 4 elementos en `px`.

Además de eso, dependiendo de la densidad de la pantalla va a ser el número de píxeles que se tengan que pintar. Para una pantalla de doble densidad (de las llamadas "retina" o "Hi-DPI") `(q/display-density)` regresa 2, entonces en efecto estamos duplicando el número de píxeles por pantalla.

Finalmente obtenemos el valor por píxel y el valor correspondiente en RGB de la paleta de colores, y aplicamos estos componentes al arreglo `px` que contiene los píxeles de pantalla, avanzando el ciclo de 4 en 4.

## Resultado

{{< doomfire >}}

Pueden mantener presionado shift para apagar el fuego temporalmente. Esta es mi parte favorita, ya que para simular que se apaga el fuego, literalmente tienes que eliminar el "combustible" poniendo el valor de la última fila en 0:

```clojure
(defn toggle-fire
  [{:keys [fire-pixels pressed-keys pixel-row pixel-count] :as state}]
  (update-in state [:fire-pixels]
             (fn [fp]
               (reduce (fn [px i] (conj px (if (< i (- pixel-count pixel-row))
                                             (get fp i)
                                             (if (contains? pressed-keys "space") 0 35))))
                       [] (range pixel-count)))))
```

## Referencias

- [How Doom fire was done](http://fabiensanglard.net/doom_fire_psx/index.html)

## Vínculos

- [Implementación completa en Github](https://github.com/cesarolea/cljs-doom-fire)
