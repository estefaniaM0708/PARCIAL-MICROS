# Generación de paisaje mediante visualización XY con ESP32-WROOM-32

En esta actividad se realizó el diseño e implementación de una **figura tipo paisaje** visualizada en un osciloscopio mediante el **modo XY**, utilizando una **ESP32-WROOM-32 programada en MicroPython**.

La figura seleccionada por el grupo corresponde a un **paisaje**, conformado por montañas de diferentes alturas, un valle, nieve, dos nubes, aves, un sol y el terreno inferior. Para generar esta imagen se emplearon dos señales independientes provenientes de la ESP32: una controla la posición horizontal **X** y la otra la posición vertical **Y** del osciloscopio.

Además, para suavizar las señales PWM generadas por la ESP32, se utilizó en cada canal un **filtro pasa bajos RC** compuesto por una **resistencia de 1 kΩ** y un **capacitor de 10 nF**.

---

# ¿Qué es una figura de Lissajous?

Las figuras de Lissajous se obtienen al utilizar dos señales para controlar simultáneamente los ejes horizontal y vertical de un osciloscopio en **modo XY**.

De manera general, este tipo de representación puede expresarse como:

[
x=A_x\sin(\omega_x t)
]

[
y=A_y\sin(\omega_y t+\delta)
]

donde (A_x) y (A_y) representan las amplitudes, (\omega_x) y (\omega_y) las frecuencias angulares y (\delta) la diferencia de fase.

En esta práctica se utiliza el mismo principio de visualización XY, pero en lugar de generar únicamente señales sinusoidales, la ESP32 recorre una serie ordenada de **coordenadas X-Y**. Al repetir este recorrido a alta velocidad, el osciloscopio reconstruye la imagen del paisaje.

El proceso general es:

```text id="q4wm6r"
Coordenadas del paisaje
          ↓
       ESP32
          ↓
    PWM X      PWM Y
      ↓          ↓
 Filtro RC    Filtro RC
      ↓          ↓
     CH1        CH2
       \        /
        \      /
      Osciloscopio
        modo XY
           ↓
        PAISAJE
```

---

# Desarrollo de la figura

La figura fue construida mediante una secuencia de coordenadas almacenadas dentro del programa. Cada coordenada representa una posición:

```text id="m5xj1m"
(x, y)
```

donde:

* `x` determina la posición horizontal.
* `y` determina la posición vertical.

El programa almacena inicialmente todos los puntos dentro de una lista:

```python id="1d28zu"
pts = []

def P(x,y):
    pts.append((x,y))
```

De esta manera, cada elemento del paisaje se convierte en un conjunto de puntos que después son recorridos rápidamente por la ESP32.

---

# Generación de líneas

Para unir dos puntos se implementó la función:

```python id="bifkik"
def L(a,b,n=10):
    for i in range(1,n+1):
        t=i/n
        P(a[0]+(b[0]-a[0])*t,
          a[1]+(b[1]-a[1])*t)
```

Esta función realiza una interpolación entre un punto inicial y uno final, generando varios puntos intermedios. Gracias a esto, las líneas se visualizan de manera continua sobre el osciloscopio.

También se utilizó la función:

```python id="u4b9hj"
def PL(v,n=10):
    for i in range(len(v)-1):
        L(v[i],v[i+1],n)
```

que permite enlazar múltiples puntos consecutivos y facilita la construcción de figuras más complejas.

---

# Construcción del paisaje

La figura se organizó en diferentes elementos para facilitar su programación: montañas, valle, nieve, nubes, aves, sol y línea del terreno.

## Montañas y valle

Los puntos principales definidos en el código son:

```python id="m3ow94"
S  = (50,174)

P1 = (70,125)
P2 = (125,165)
V  = (165,78)
P3 = (205,145)
P4 = (230,112)
```

Cada uno de estos puntos representa una parte clave del paisaje.

| Punto | Función           |
| ----- | ----------------- |
| `P1`  | Montaña izquierda |
| `P2`  | Montaña más alta  |
| `V`   | Valle             |
| `P3`  | Montaña derecha   |
| `P4`  | Montaña menor     |

Para generar el valle, el programa realiza una bajada pronunciada desde una de las montañas:

```python id="tpxspn"
PL((
    P3,
    (194,120),
    (180,92),
    V
),10)
```

Posteriormente se realiza el ascenso desde el valle hacia la montaña más alta:

```python id="r6vyy7"
PL((
    V,
    (176,95),
    (153,125),
    P2
),10)
```

Esto permite que la figura no sea plana y que tenga una variación visible de alturas.

---

# Nieve en las montañas

Para mejorar el aspecto visual del paisaje se añadió nieve en la cima de las montañas mediante la función:

```python id="9bq2k7"
def nieve(p,a,b,c):
    PL((p,a,b,c,p),6)
```

Esta función crea una forma pequeña e irregular alrededor de la cima. Por ejemplo, en la montaña principal se utilizó:

```python id="5hkb9w"
nieve(P2,
      (115,147),
      (125,155),
      (136,144))
```

---

# Nubes

Las nubes se construyeron mediante una secuencia de puntos relativos a un centro:

```python id="m9ajwn"
def nube(cx,cy):
    v=[(-25,0),(-20,10),(-10,13),(-4,22),(7,18),
       (14,25),(26,17),(29,5),(20,-3),(5,-4),
       (-10,-3),(-25,0)]

    PL([(cx+a,cy+b) for a,b in v],5)
```

En el paisaje final se agregaron **dos nubes**, ubicadas en diferentes posiciones para dar mayor equilibrio a la composición.

---

# Sol

El sol se construyó con una circunferencia y varios rayos, utilizando funciones trigonométricas:

```python id="81jv6d"
def sol(cx,cy,r=18):
```

En esta función se usan seno y coseno para recorrer distintos ángulos y así dibujar tanto el contorno circular como los rayos del sol.

---

# Aves

Las aves se representaron mediante pequeñas líneas con forma de vuelo:

```python id="5thhum"
def ave(cx,cy,s=8):
    PL(((cx-s,cy+s//2),
        (cx,cy-s//2),
        (cx+s,cy+s)),5)
```

En el diseño final se añadieron dos aves en la parte superior del paisaje.

---

# Generación de señales con la ESP32-WROOM-32

Para controlar los ejes X y Y del osciloscopio se utilizaron dos salidas PWM de la ESP32:

```python id="23bigb"
X = PWM(Pin(4), freq=500_000)
Y = PWM(Pin(5), freq=500_000)
```

Las conexiones empleadas fueron:

| ESP32 | Función          | Osciloscopio |
| ----- | ---------------- | ------------ |
| GPIO4 | Coordenada X     | CH1 / X      |
| GPIO5 | Coordenada Y     | CH2 / Y      |
| GND   | Referencia común | GND          |

La frecuencia PWM configurada en ambos canales fue de:

```text id="jlwmcc"
500 kHz
```

La ESP32 no entrega directamente una señal analógica pura desde estos pines, sino una señal PWM cuyo ciclo útil cambia de acuerdo con el valor de la coordenada.

---

# Conversión de coordenadas a PWM

Las coordenadas del dibujo se encuentran aproximadamente en un rango de:

```text id="6l7a79"
0 a 255
```

Para convertirlas en valores adecuados para `duty_u16()`, se implementó:

```python id="4enhly"
def D(v,inv=False):
    v=max(0,min(255,v))

    if inv:
        v=255-v

    return int(2500+v*(63000-2500)/255)
```

Esta función realiza tres tareas principales:

1. Limita los valores al intervalo permitido.
2. Invierte el eje si es necesario.
3. Escala la coordenada a un valor compatible con el PWM.

Finalmente, los datos se preparan con:

```python id="q1y5d2"
datos=[(D(x,True),D(y)) for x,y in pts]
```

En este caso se invierte el eje X para conservar la orientación correcta del paisaje en la pantalla del osciloscopio.

---

# Filtro pasa bajos en cada canal

Dado que las señales generadas por la ESP32 son PWM, fue necesario usar un **filtro pasa bajos RC independiente para cada salida**.

En ambos canales se utilizó:

```text id="czfcn5"
R = 1 kΩ
C = 10 nF
```

La conexión para cada canal es:

```text id="jkul3n"
GPIO ESP32
    │
   1 kΩ
    │
    ├────────→ Canal del osciloscopio
    │
  10 nF
    │
   GND
```

Por tanto, el montaje completo es:

```text id="lnbdum"
GPIO4 ── 1 kΩ ──┬── CH1 / X
                 │
               10 nF
                 │
                GND


GPIO5 ── 1 kΩ ──┬── CH2 / Y
                 │
               10 nF
                 │
                GND
```

## Frecuencia de corte

La frecuencia de corte de un filtro RC se calcula mediante:

[
f_c=\frac{1}{2\pi RC}
]

Sustituyendo:

[
R=1000\ \Omega
]

[
C=10\times10^{-9}\ F
]

se obtiene:

[
f_c=\frac{1}{2\pi(1000)(10\times10^{-9})}
]

[
\boxed{f_c\approx 15.9\ kHz}
]

Como la señal PWM trabaja a **500 kHz**, esta frecuencia es mucho mayor que la frecuencia de corte del filtro. Por ello, la componente pulsante del PWM se atenúa considerablemente y se obtiene una señal más suave en cada canal.

En términos prácticos:

```text id="tcjlwm"
PWM de 500 kHz
      ↓
Filtro RC
      ↓
Señal suavizada
      ↓
Posición X o Y
```

Sin estos filtros, la visualización tendría mucho más ruido y la figura sería menos estable.

---

# Recorrido de la figura

Una vez generados todos los puntos, el programa entra en un ciclo infinito:

```python id="908578"
while True:
    for x,y in datos:
        X.duty_u16(x)
        Y.duty_u16(y)
        sleep_us(T)
```

El tiempo empleado entre cada punto es:

```python id="2psfqr"
T = 25
```

es decir, aproximadamente **25 μs por punto**.

Al repetirse este recorrido continuamente, el osciloscopio reconstruye la figura completa gracias a la rapidez de actualización y a la persistencia visual de la pantalla.

---

# Uso de la ESP32-WROOM-32

La **ESP32-WROOM-32** fue el elemento central de la práctica. Su función fue ejecutar el programa, recorrer la secuencia de coordenadas y generar las dos señales PWM necesarias para el modo XY.

Los recursos de la placa utilizados en esta práctica fueron:

| Recurso     | Uso                                       |
| ----------- | ----------------------------------------- |
| Procesador  | Ejecuta el recorrido de coordenadas       |
| GPIO4       | Señal del eje X                           |
| GPIO5       | Señal del eje Y                           |
| PWM         | Generación de niveles variables           |
| Memoria     | Almacenamiento de los puntos de la figura |
| MicroPython | Programación del sistema                  |

Aunque la tarjeta posee otros periféricos como Wi-Fi, Bluetooth y ADC, en esta práctica no fueron necesarios.

---

# Arquitectura desarrollada

La arquitectura implementada puede representarse de la siguiente forma:

```text id="x6vp4v"
                CÓDIGO MICROPYTHON
                       │
                       ↓
              Coordenadas X - Y
                       │
                       ↓
                ESP32-WROOM-32
                 ┌─────┴─────┐
                 │           │
              GPIO4       GPIO5
               PWM X       PWM Y
                 │           │
                 ↓           ↓
              R = 1kΩ     R = 1kΩ
                 │           │
             C = 10nF     C = 10nF
                 │           │
                 ↓           ↓
                CH1         CH2
                 │           │
                 └─────┬─────┘
                       ↓
                  OSCILOSCOPIO
                     MODO XY
                       │
                       ↓
                    PAISAJE
```

Esta arquitectura muestra claramente tres etapas:

1. **Procesamiento:** la ESP32 calcula y recorre los puntos.
2. **Filtrado:** el PWM se suaviza mediante filtros RC.
3. **Visualización:** el osciloscopio reconstruye la figura en modo XY.

---

# Funcionamiento real en el osciloscopio

Para la prueba física se conectó:

```text id="qe421h"
GPIO4 → filtro RC → CH1
GPIO5 → filtro RC → CH2
GND ESP32 → GND del osciloscopio
```

Posteriormente el osciloscopio se configuró en:

```text id="e8jyas"
Modo XY
X = CH1
Y = CH2
```

Con esta configuración fue posible visualizar el paisaje compuesto por montañas, valle, terreno, nubes, aves y sol.

La evidencia experimental demuestra que ambas señales están correctamente sincronizadas y que el uso de los filtros RC permite obtener una figura suficientemente estable y reconocible.

### Evidencia del funcionamiento real

En este apartado se adjunta la fotografía del resultado observado en el osciloscopio:

```markdown id="aukoqf"
![Paisaje visualizado en el osciloscopio](imagenes/paisaje_osciloscopio.jpg)
```

---

# Resultado

Como resultado se logró representar un **paisaje completo mediante visualización XY utilizando una ESP32-WROOM-32**.

La figura fue construida a partir de coordenadas organizadas en el programa, las cuales se transforman en señales PWM para los dos ejes del osciloscopio. Los filtros pasa bajos de **1 kΩ y 10 nF** fueron esenciales para suavizar estas señales antes de ingresarlas al osciloscopio.

La visualización obtenida permitió distinguir claramente los principales elementos del paisaje: montañas, valle, nieve, nubes, sol y aves.

---

# Conclusión

Con el desarrollo de esta actividad se logró integrar programación en MicroPython, generación de PWM, filtrado analógico y visualización mediante osciloscopio.

La **ESP32-WROOM-32** se encargó de recorrer las coordenadas del paisaje y generar dos señales independientes, una para cada eje. Los filtros RC implementados con **resistencia de 1 kΩ** y **capacitor de 10 nF** permitieron suavizar las señales PWM y hacer posible una representación más estable en el modo XY.

La práctica permitió comprender que el osciloscopio puede utilizarse no solo para observar formas de onda convencionales, sino también para representar trayectorias diseñadas mediante coordenadas, siempre que exista sincronización entre ambos canales.

---

# Recursos científicos y datos empleados

## Recursos científicos

* **Espressif Systems**, documentación técnica de la **ESP32-WROOM-32**.
* **MicroPython Documentation**, referencia del módulo `machine` y del uso de `PWM`.
* Fundamento teórico de **figuras de Lissajous** y visualización en modo XY.
* Fundamento teórico de **filtros RC pasa bajos**.
* Ecuación de frecuencia de corte:

[
f_c=\frac{1}{2\pi RC}
]
