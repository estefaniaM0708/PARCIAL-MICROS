# Generación de paisaje mediante visualización XY con ESP32-WROOM-32

En esta actividad se realizó el diseño e implementación de una **figura tipo paisaje** visualizada en un osciloscopio mediante el **modo XY**, utilizando una **ESP32-WROOM-32 programada en MicroPython**.

La figura seleccionada por el grupo corresponde a un paisaje compuesto por **montañas de diferentes alturas, un valle, nieve, dos nubes, aves, un sol y el terreno inferior**. Para generar esta imagen se utilizaron dos señales provenientes de la ESP32: una encargada de controlar la posición horizontal **X** y otra para la posición vertical **Y** del osciloscopio.

Además, como las señales generadas por la ESP32 corresponden a PWM, se implementó en cada canal un **filtro pasa bajos RC**, formado por una resistencia de **1 kΩ** y un capacitor de **10 nF**. Estos filtros permiten suavizar las señales antes de ingresarlas al osciloscopio.

---

# ¿Qué es una figura de Lissajous?

Las figuras de Lissajous se obtienen al utilizar dos señales para controlar simultáneamente los ejes horizontal y vertical de un osciloscopio configurado en **modo XY**.

De forma general, estas señales pueden representarse como:

**x = Aₓ · sin(ωₓ · t)**

**y = Aᵧ · sin(ωᵧ · t + δ)**

Donde:

* **Aₓ y Aᵧ:** amplitudes de las señales de los ejes X y Y.
* **ωₓ y ωᵧ:** frecuencias angulares.
* **t:** tiempo.
* **δ:** diferencia de fase entre las señales.

En esta práctica se aprovecha el principio de visualización XY, pero en lugar de utilizar únicamente señales sinusoidales, la ESP32 recorre una serie ordenada de **coordenadas X-Y**. Al repetir este recorrido rápidamente, el osciloscopio reconstruye visualmente el paisaje.

El funcionamiento general es:

```text
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

La figura fue construida mediante una secuencia de coordenadas almacenadas dentro del programa. Cada coordenada representa una posición determinada del dibujo:

```text
(x, y)
```

Donde:

* **x:** determina la posición horizontal.
* **y:** determina la posición vertical.

Los puntos se almacenan inicialmente en una lista:

```python
pts = []

def P(x,y):
    pts.append((x,y))
```

De esta forma, los diferentes elementos del paisaje se convierten en conjuntos de puntos que posteriormente son recorridos por la ESP32.

---

# Generación de líneas

Para unir dos coordenadas se desarrolló la función:

```python
def L(a,b,n=10):
    for i in range(1,n+1):
        t=i/n
        P(a[0]+(b[0]-a[0])*t,
          a[1]+(b[1]-a[1])*t)
```

Esta función genera diferentes puntos intermedios entre un punto inicial y uno final. Esto evita movimientos demasiado bruscos y permite obtener líneas más continuas en la pantalla del osciloscopio.

También se utiliza:

```python
def PL(v,n=10):
    for i in range(len(v)-1):
        L(v[i],v[i+1],n)
```

Esta función permite conectar varias coordenadas consecutivamente, facilitando la creación de formas más complejas como las montañas, el valle y el terreno.

---

# Construcción del paisaje

El paisaje se dividió en diferentes elementos para facilitar su programación. Dentro del código se encuentran las montañas, el valle, la nieve, las nubes, las aves, el sol y el terreno inferior.

## Montañas y valle

Los puntos principales definidos son:

```python
S  = (50,174)

P1 = (70,125)
P2 = (125,165)
V  = (165,78)
P3 = (205,145)
P4 = (230,112)
```

Cada uno representa una parte importante de la figura:

| Punto | Función                 |
| ----- | ----------------------- |
| `P1`  | Montaña izquierda       |
| `P2`  | Montaña de mayor altura |
| `V`   | Valle                   |
| `P3`  | Montaña derecha         |
| `P4`  | Montaña de menor altura |

El valle se genera realizando una bajada desde una de las montañas:

```python
PL((
    P3,
    (194,120),
    (180,92),
    V
),10)
```

Posteriormente se realiza el ascenso desde el valle hacia la siguiente montaña:

```python
PL((
    V,
    (176,95),
    (153,125),
    P2
),10)
```

Esto permite obtener una cordillera con diferentes alturas y una separación clara entre las montañas.

---

# Nieve en las montañas

Para dar mayor detalle al paisaje se agregó nieve en las cimas mediante:

```python
def nieve(p,a,b,c):
    PL((p,a,b,c,p),6)
```

Por ejemplo, para una de las montañas se emplea:

```python
nieve(P2,
      (115,147),
      (125,155),
      (136,144))
```

Estos pequeños recorridos generan formas irregulares sobre las cimas y permiten diferenciarlas del resto de la montaña.

---

# Nubes

Las nubes se generaron mediante diferentes coordenadas relativas a una posición central:

```python
def nube(cx,cy):
    v=[(-25,0),(-20,10),(-10,13),(-4,22),(7,18),
       (14,25),(26,17),(29,5),(20,-3),(5,-4),
       (-10,-3),(-25,0)]

    PL([(cx+a,cy+b) for a,b in v],5)
```

En el paisaje final se incorporaron **dos nubes** en diferentes posiciones, buscando que la imagen tuviera una distribución más completa y menos simétrica.

---

# Sol

El sol se genera mediante una circunferencia acompañada de varios rayos.

```python
def sol(cx,cy,r=18):
```

Para construirlo se utilizan las funciones trigonométricas seno y coseno, que permiten recorrer diferentes posiciones alrededor de un punto central.

Cada punto del círculo se obtiene variando el ángulo y calculando sus coordenadas horizontales y verticales. Después se añaden líneas hacia el exterior para representar los rayos.

---

# Aves

Las aves se representan mediante pequeñas líneas similares a la forma de un ave en vuelo:

```python
def ave(cx,cy,s=8):
    PL(((cx-s,cy+s//2),
        (cx,cy-s//2),
        (cx+s,cy+s)),5)
```

En el diseño final se incorporaron **dos aves** en la zona superior del paisaje.

---

# Generación de señales con la ESP32-WROOM-32

Para controlar los dos ejes del osciloscopio se utilizaron dos salidas PWM:

```python
X = PWM(Pin(4), freq=500_000)
Y = PWM(Pin(5), freq=500_000)
```

Las conexiones utilizadas son:

| ESP32-WROOM-32 | Función          | Osciloscopio |
| -------------- | ---------------- | ------------ |
| GPIO4          | Señal del eje X  | CH1 / X      |
| GPIO5          | Señal del eje Y  | CH2 / Y      |
| GND            | Referencia común | GND          |

La frecuencia establecida para ambos PWM es:

**fPWM = 500 kHz**

La ESP32 genera una señal digital de alta frecuencia y modifica su ciclo útil dependiendo de la coordenada que debe representarse.

De esta forma, un valor diferente de ciclo útil produce un nivel promedio de voltaje diferente después del filtro.

---

# Conversión de coordenadas a PWM

Las coordenadas utilizadas en el programa se manejan aproximadamente entre:

**0 y 255**

Para transformar estos valores en datos compatibles con `duty_u16()` se utiliza:

```python
def D(v,inv=False):
    v=max(0,min(255,v))

    if inv:
        v=255-v

    return int(2500+v*(63000-2500)/255)
```

Esta función realiza tres procesos:

1. Limita el valor de la coordenada entre 0 y 255.
2. Permite invertir el eje cuando sea necesario.
3. Convierte la coordenada al rango utilizado por `duty_u16()`.

Finalmente, las coordenadas se transforman mediante:

```python
datos=[(D(x,True),D(y)) for x,y in pts]
```

En este caso se utiliza `True` para invertir el eje X y conservar la orientación correcta del paisaje en el osciloscopio.

---

# Filtro pasa bajos en cada canal

Las salidas GPIO4 y GPIO5 generan señales PWM. Para obtener una señal más suave y adecuada para el osciloscopio se utilizó un **filtro pasa bajos RC independiente en cada canal**.

Los componentes utilizados fueron:

**R = 1 kΩ**

**C = 10 nF**

La conexión de cada filtro es:

```text
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

Por lo tanto, para los dos canales se tiene:

```text
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

---

# Cálculo de la frecuencia de corte

La frecuencia de corte de un filtro pasa bajos RC se determina mediante:

**fc = 1 / (2πRC)**

Para esta práctica se utilizaron:

**R = 1000 Ω**

**C = 10 nF = 10 × 10⁻⁹ F**

Reemplazando los valores:

**fc = 1 / [2π · 1000 · (10 × 10⁻⁹)]**

Por lo tanto:

**fc ≈ 15 915 Hz**

o aproximadamente:

**fc ≈ 15,9 kHz**

La señal PWM generada por la ESP32 trabaja a:

**fPWM = 500 kHz**

Por tanto:

**500 kHz > 15,9 kHz**

La frecuencia del PWM se encuentra muy por encima de la frecuencia de corte del filtro. Esto permite atenuar gran parte de las componentes de alta frecuencia asociadas a la conmutación del PWM y conservar principalmente su valor promedio.

El proceso puede resumirse como:

```text
PWM de 500 kHz
      ↓
Filtro RC
fc ≈ 15,9 kHz
      ↓
Señal suavizada
      ↓
Posición X o Y
```

Esto mejora la estabilidad y definición del paisaje observado en el osciloscopio.

---

# Recorrido de la figura

Después de generar y convertir todas las coordenadas, el programa comienza a recorrerlas continuamente:

```python
while True:
    for x,y in datos:
        X.duty_u16(x)
        Y.duty_u16(y)
        sleep_us(T)
```

El tiempo utilizado entre cada punto es:

**T = 25 μs**

Esto significa que cada pareja de coordenadas permanece aproximadamente 25 microsegundos antes de pasar a la siguiente.

El recorrido se repite constantemente. Debido a la rapidez con la que se actualizan los puntos, el osciloscopio muestra el conjunto como una figura completa y aparentemente estable.

---

# Uso de la ESP32-WROOM-32

La **ESP32-WROOM-32** es el elemento principal encargado de generar las señales necesarias para construir el paisaje.

Durante esta práctica se utilizaron los siguientes recursos:

| Recurso     | Aplicación                                        |
| ----------- | ------------------------------------------------- |
| Procesador  | Ejecución del programa y recorrido de coordenadas |
| GPIO4       | Generación de la señal para X                     |
| GPIO5       | Generación de la señal para Y                     |
| PWM         | Representación de las coordenadas                 |
| Memoria     | Almacenamiento de los puntos                      |
| MicroPython | Desarrollo del programa                           |

En esta aplicación no fue necesario utilizar otros periféricos disponibles en la ESP32, como Wi-Fi, Bluetooth o ADC.

---

# Arquitectura desarrollada

La arquitectura del sistema implementado puede representarse como:

```text
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
              R = 1 kΩ    R = 1 kΩ
                 │           │
             C = 10 nF   C = 10 nF
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

La arquitectura puede dividirse en tres etapas principales:

### 1. Procesamiento

La ESP32 ejecuta el código de MicroPython y recorre todas las coordenadas correspondientes a la figura.

### 2. Generación y filtrado

Los GPIO4 y GPIO5 generan señales PWM de 500 kHz. Posteriormente cada señal pasa por un filtro RC de **1 kΩ y 10 nF**.

### 3. Visualización

Las señales filtradas ingresan a CH1 y CH2 del osciloscopio. Al activar el modo XY, cada pareja de voltajes representa una posición dentro de la pantalla y permite reconstruir el paisaje.

---

# Funcionamiento real en el osciloscopio

Para realizar la prueba física se conectaron las señales de la siguiente forma:

```text
GPIO4 → filtro RC → CH1
GPIO5 → filtro RC → CH2
GND ESP32 → referencia GND
```

El osciloscopio se configuró posteriormente en:

```text
Modo XY
X = CH1
Y = CH2
```

Con esta configuración fue posible visualizar el paisaje generado mediante el código.

En la pantalla se pueden reconocer las **montañas de diferentes alturas, el valle, las nubes, el sol, las aves y el terreno inferior**.

La visualización obtenida demuestra que los dos canales se encuentran sincronizados y que el filtrado realizado sobre las señales PWM permite obtener una figura reconocible y estable.

---

# Resultado

Como resultado se logró generar un **paisaje completo utilizando una ESP32-WROOM-32 y un osciloscopio en modo XY**.

La imagen fue construida mediante coordenadas definidas dentro del programa. Estas coordenadas fueron transformadas en ciclos útiles PWM y enviadas simultáneamente mediante GPIO4 y GPIO5.

Los filtros pasa bajos formados por **R = 1 kΩ y C = 10 nF** permitieron suavizar las señales antes de introducirlas al osciloscopio. Para estos componentes se obtuvo una frecuencia de corte aproximada de:

**fc ≈ 15,9 kHz**

mientras que la frecuencia PWM utilizada fue de:

**fPWM = 500 kHz**

Finalmente, al configurar el osciloscopio en modo XY se logró visualizar correctamente la figura diseñada.

---

# Conclusión

Con el desarrollo de esta actividad se logró integrar **programación en MicroPython, generación de señales PWM, filtrado analógico y visualización mediante un osciloscopio**.

La ESP32-WROOM-32 se encargó de recorrer las coordenadas que conforman el paisaje y generar dos señales independientes para controlar los ejes X y Y.

Los filtros pasa bajos implementados con resistencias de **1 kΩ** y capacitores de **10 nF** permitieron reducir las componentes de alta frecuencia del PWM y obtener señales más suaves para la entrada del osciloscopio.

La práctica permitió comprobar que el modo XY puede utilizarse no solamente para visualizar figuras de Lissajous convencionales, sino también para representar trayectorias diseñadas mediante coordenadas. De esta manera fue posible obtener un paisaje compuesto por diferentes elementos utilizando únicamente dos señales sincronizadas.

---

# Recursos científicos y datos empleados

## Recursos científicos

* **Espressif Systems – ESP32-WROOM-32 Datasheet.**
  Información técnica relacionada con la arquitectura, GPIO y características eléctricas del módulo.
  https://documentation.espressif.com/esp32-wroom-32_datasheet_en.html

* **MicroPython – Quick reference for the ESP32.**
  Documentación utilizada como referencia para el manejo de GPIO y PWM mediante MicroPython.
  https://docs.micropython.org/en/latest/esp32/quickref.html

* **MicroPython – machine.PWM.**
  Referencia de la clase utilizada para la generación de las señales PWM.
  https://docs.micropython.org/en/latest/library/machine.PWM.html

* **Repositorio de referencia – Figuras de Lissajous.**
  Material suministrado como guía para el desarrollo y documentación de la actividad.
  https://github.com/dialejobv/aplicacion_sistemas_embebidos/blob/main/2)%20LABORATORIO/Explicaci%C3%B3n_Fig_Lissajous.md

## Datos empleados

| Parámetro                | Valor          |
| ------------------------ | -------------- |
| Microcontrolador         | ESP32-WROOM-32 |
| Lenguaje de programación | MicroPython    |
| Eje X                    | GPIO4          |
| Eje Y                    | GPIO5          |
| Frecuencia PWM           | 500 kHz        |
| Tiempo entre puntos      | 25 μs          |
| Resistencia del filtro X | 1 kΩ           |
| Capacitor del filtro X   | 10 nF          |
| Resistencia del filtro Y | 1 kΩ           |
| Capacitor del filtro Y   | 10 nF          |
| Frecuencia de corte      | ≈ 15,9 kHz     |
| Canal X del osciloscopio | CH1            |
| Canal Y del osciloscopio | CH2            |
| Configuración            | Modo XY        |
| Figura desarrollada      | Paisaje        |

