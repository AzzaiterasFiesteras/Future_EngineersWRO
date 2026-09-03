
# Funciones del código 

| **Funciones** |  **Sintaxis** | **Uso explicado** | 
| :--- | :---: | ---:| 
| `setup()`|`void setup()`  |Se ejecuta una sola vez al encender la placa. Inicializa comunicación del puerto serie, configura pines, conecta el servo, inicia el sensor BNO055 y calibra el giroscopio| 
| `loop()`|`void loop()`  |Se ejecuta continuamente. Lee sensores ultrasónicos, detecta obstáculos, controla el movimiento, corrige dirección y cuenta los giros para completar las vueltas.| 
| `obtenerGrados()`|`float obtenerGrados() ` |Lee la orientación del sensor BNO055 y devuelve el ángulo en grados respecto al eje X `(orientation.x)`. Se usa para saber hacia dónde apunta el robot.| 
| `prepararSensor()`|`void prepararSensor() ` |Espera hasta que el giroscopio tenga máxima precisión `(gyro == 3)`. Después guarda el ángulo inicial como referencia en `anguloObjetivo.`| 
| `hacerGiro()`|`void hacerGiro(int grados) ` |Ajusta el objetivo angular y mueve el servo para girar el coche. El robot sigue girando hasta que el error sea menor a 5°.| 
| `terminarCarrera()`|`void terminarCarrera() ` |Hace que el coche avance un poco más para no seguir girando 8para que no llegue a chocarse con la pared interna) y luego apaga el motor.| 

## Funciones dentro del `void setup`

| **Funciones** | **Uso explicado** | 
| :--- | ---: |
| `Serial.begin(9600)`| Inicia comunicación del puerto serie. |
| `pinMode()`| Configura pines como entrada o salida |
| `cochino.attach(pinservo)`| Conecta el servo |
| `cochino.write(centroServo)`|`Coloca el servo centrado ` |
| `bno.begin()`|Inicializa el sensor BNO055 |
| `prepararSensor()`|Calibra el giroscopio |

## Funciones de librerías

| **Función** |**Librería** | **Uso explicado** | 
| :--- | :---: | ---:| 
| `Serial.begin()`|Arduino  |Comunicación con el puerto serie| 
| `Serial.print()`|Arduino  |Mostrar texto en el puerto serie.| 
| `Serial.println()`|Arduino |Mostrar texto con salto en el puerto serie.| 
| `pinMode()`| Arduino|Configura pines| 
| `digitalWrite()`|Arduino |Encender/apagar pin.| 
| `digitalRead()` |Arduino |Leer pin digital.| 
| `analogWrite()`|Arduino  |PWM para velocidad| 
| `delay()`|Arduino  |Pausa del programa.| 
| `abs()`|Arduino/C++ |Valor absoluto.| 
| `attach()` | Servo | Conectar servo|
| `write()`  | Servo | Mover servo |
| `begin()`| BNO055 | Iniciar sensor  |


# Explicación del programa - Open Challenge
## Variables y librerías
```C++
#include <Wire.h>
#include <Adafruit_Sensor.h>
#include <Adafruit_BNO055.h>
#include <Servo.h>
#include <Ultrasonic.h>

int ENA = 5;
int IN1 = 6;
int IN2 = 7;

Servo cochino;
Adafruit_BNO055 bno = Adafruit_BNO055(55);

const int pinservo = 9;
const int pinPulsador=13;

float anguloObjetivo;  
int centroServo = 35;  // Posición recta
int sentido = 0;       // 1 para derecha, 2 para izquierda
int contadorGiros = 0; // Control de esquinas para las 3 vueltas
int contador1=0;
int contador2=0;
bool start = false; 
```
Antes de desarrollar la lógica de la programación, lo primero fue instalar las librerías tanto de los motores como del servomotor, de los sensores de ultrasonidos y del giroscopio para facilitar la programación mediante `#include <nombre de la librería>`Y además, nombrar el servo y el giroscopio. Los sensores de ultrasonidos los nombramos posteriormente. Y por último, antes de nombrar a los sensores, definimos todas las variables:
- `int ENA = 5`; `int IN1 = 6`; `int IN2 = 7`; establecemos los pines a los que van conectados los motores en el escudo L298N.
- `const int pinservo = 9;` establecemos el pin al que está conectado el servo. 
- `float anguloObjetivo:` que es el ángulo al que queremos ir.
- `int centroServo = 35; ` que es la posición del servomotor en la cual el sistema de dirección del robot está orientado para ir recto.
- `int sentido = 0; ` el robot identificará el sentido al que debe girar en el primer giro. Para esto era necesario crear una variable para la cual el valor 1 sea girar hacia la derecha y el valor 2 para la izquierda. Es por esto, que desde el principio del código establecemos que sea 0.
- `int contadorGiros = 0;` gracias a esta variable que cuenta giros, el robot es capaz de pararse al completar las 3 vueltas (12 giros en total).
- `contador1` y `contador2`: estos contadores se utilizan para asegurar que la situación no sea una lectura aleatoria o temporal, sino algo que persista en el tiempo. En otras palabras, es una forma de confirmar la fiabilidad de los datos del sensor. Básicamente, el robot no reacciona a una sola medición. En cambio, espera a que el problema se repita varias veces y solo entonces responde deteniéndose para evitar colisiones o continuar en una situación peligrosa.
- `bool start = false`: Este tipo de variable solo puede tener dos valores: true (verdadero) o false (falso). En este caso empieza en false, lo que significa que el sistema está apagado. Lo usaremos para iniciar el programa desde el pulsador.
  
```C++
#define TRIG_IZQ 8
#define ECHO_IZQ 10
Ultrasonic ultraIZQ(TRIG_IZQ, ECHO_IZQ, 60000);
#define TRIG_CENTRO 3
#define ECHO_CENTRO 4
Ultrasonic ultraCENTRO(TRIG_CENTRO, ECHO_CENTRO, 60000);
#define TRIG_DERECH 11
#define ECHO_DERECH 12
Ultrasonic ultraDERECH(TRIG_DERECH, ECHO_DERECH, 60000);
```

Como ya sabemos, los sensores de ultrasonidos están compuestos por un `trigger` (que emite una señal) y un `echo` (que recibe esa señal). Según el tiempo que tarda en rebotar esa señal podemos calcular la distancia. Para esto, tuvimos que indicar en la programación los pines a los que estaban conectados cada trigger y cada echo de cada sensor, de tal manera que:
- **Trigger** (pin 8) y **echo** (pin 10) = sensor ultrasonidos izquierda
- **Trigger** (pin 3) y **echo** (pin 4) = sensor ultrasonidos centro
- **Trigger** (pin 11) y **echo** (pin 12) = sensor ultrasonidos derecha
  
## Void setup

```C++
void setup() {
  Serial.begin(9600);
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(ENA, OUTPUT);
  pinMode(pinPulsador, INPUT_PULLUP);

  cochino.attach(pinservo);
  cochino.write(centroServo);
  
  Serial.println("Hola, iniciando sistema...");

  if (!bno.begin()) {
    Serial.println("No se encuentra el sensor BNO");
    while (1); // Bloqueo si no hay sensor
  }
  prepararSensor(); 
  
}
```
El` void setup` es una función que se crea al principio del programa y sólo se ejecuta una única vez al comienzo del programa. Su propósito principal es iniciar todos los componentes que se necesitan para comenzar a funcionar correctamente. En nuestro caso, activamos el puerto serie para poder calibrar las lecturas de los sensores de ultrasonido, y configuramos el pin del servomotor para que pueda empezar a funcionar desde el inicio. Además iniciamos los motores y el pulsador. A parte, añadimos un condicional para que en el caso de que no se detecte el giroscopio que lo imprima en el puerto serie y también bloquee el sistema. Además, llamamos a la función que calibra el giroscopio para que el robot sepa dónde está el "norte".  

## Void loop

```C++
  if (digitalRead(pinPulsador) == LOW) {
    start = true;
    Serial.print(2);
  }
  if (start == true) {
  // Lógica de fin de carrera: tras 12 giros (3 vueltas), el robot para
    if (contadorGiros >= 12) {
      terminarCarrera();
      while(1); 
    }
```
En la primera parte del void loop hemos establecido la programación del pulsador de tal manera que al pulsarlo empiece todo el programa y además, imprima en el puerto serie el número 2 para comprobar que el pulsador está funcionando. Además, añadimos la lógica del final: tras 12 giros, es decir, tres vueltas el robot se detiene. 

``` C++
    int cmCentro = ultraCENTRO.Ranging(CM);
    float anguloAhora = obtenerGrados();
    int cmIzq = ultraIZQ.Ranging(CM);
    int cmDer = ultraDERECH.Ranging(CM);
```
Aquí leemoos los datos de los sensores y los guardamos en variables para usarlos depués. Primero establecemos las distancias de los tres sensores de ultrasonidos, es decir, que lo que mida el sensor va a ser la distancia expresada en centímetros. Después, la variable anguloAhora guarda el resultado de la función obtenerGrados(). Esta función devuelve el ángulo actual del robot, y se guarda como un número decimal (`float`) porque puede tener decimales.

``` C++
  if (cmCentro < 80) {
      contador1++;
    } else {
      contador1=0;
    }
    if (cmCentro < 140 && abs(cmIzq-cmDer)>120) {
      contador2++;
    } else {
      contador2=0;
    }
if (contador1>2 || contador2>2) {
      contador1=0;
      contador2=0;
      digitalWrite(IN1, LOW);
      digitalWrite(IN2, LOW);
      analogWrite(ENA, 0);
      delay(500);
```
En estas funciones el programa está usando dos contadores para detectar cuando el robot necesitaría girar. Primero, el robot va comprobando las distancias con sus sensores. Si detecta que se acerca a las paredes varias veces seguidas, o si detecta una situación rara donde hay mucha diferencia entre lo que ve a la izquierda y a la derecha, empieza a sumar en unos contadores. Estos contadores sirven para asegurarse de que no es algo puntual, sino que realmente el problema se mantiene durante un tiempo. Es básicamente una manera de asegurar que las mediciones de los sensores son correctas. En otras palabras, el robot no reacciona a un solo dato, sino que espera a que el problema se repita varias veces, y entonces se para para evitar chocar o seguir en una mala situación.

``` C++
     if (contadorGiros == 0) {
        if (cmIzq < cmDer) sentido = 1; 
        else sentido = 2;              
        Serial.print("Sentido del circuito: ");
        Serial.println(sentido);
      }
      if (sentido == 1) hacerGiro(90);
      else hacerGiro(-90);

      contadorGiros++; 
      Serial.print("Esquina número: ");
      Serial.println(contadorGiros);
    }
```
Es entonces, cuando en la primera pared decidimos si el circuito se hace por la izquierda o por la derecha. Además, establecemos que si la distancia de la izquierda es menor que la de la derecha, entonces el sentido va a ser el 1; el correspondiente a la derecha. Y por lo tanto, si es al revés el sentido va a ser el 2; el que corresponde a la izquierda. Por lo tanto para la ejecución del giro decimos que si el sentido es el 1 (derecha) entonces que el robot haga un giro de 90 grados. Y si es al revés, que haga un giro de -90 grados para girar a la izquierda (sentido 2). Además, conforme vamos haciendo los giros los vamos contando ya que posteriormente los necesitaremos para parar el robot. Para asegurarnos de que el contador de giros está funcionando, mostramos en el puerto serie el número de esquinas, y por tanto, de giros. 

``` C++
else {
      digitalWrite(IN1, LOW);
      digitalWrite(IN2, HIGH);
      analogWrite(ENA, 200);

      // Ajuste de dirección para ir siempre rectos
      float diferencia = anguloObjetivo - anguloAhora;
      if (diferencia > 180) diferencia -= 360;
      if (diferencia < -180) diferencia += 360;

      // Si nos desviamos, el servo corrige automáticamente
      if (diferencia > 3) {
        cochino.write(centroServo + 12);
      } else if (diferencia < -3) {
        cochino.write(centroServo - 12);
      } else {
        cochino.write(centroServo);
      }
    } 
  }
}
```


Sin embargo, si el robot no detecta ninguna pared, significa que el camino está libre, por lo cual activamos los motores y aumentamos su velocidad y además ajustamos la dirección para que siempre vaya recto y por tanto si el robot se desvía, el servo se corrige automáticamente.

## Funciones propias 
``` C++
float obtenerGrados() {
  sensors_event_t datos;
  bno.getEvent(&datos);
  return datos.orientation.x;
}
```
Este código sirve para que el robot pueda saber hacia dónde está girado, calibrarse y hacer giros precisos usando un sensor BNO055 (un giroscopio). Esta lee el sensor de orientación del robot. El sensor le dice en qué dirección está mirando (como una brújula). La función guarda esos datos y devuelve el ángulo actual del robot, es decir, cuántos grados está girado.
```C++
void prepararSensor() {
  uint8_t sys, gyro, accel, mag;
  Serial.println("Esperando calibración del giroscopio...");
  do {
    bno.getCalibration(&sys, &gyro, &accel, &mag);
  } while (gyro < 3); // Nivel 3 es el máximo de precisión
  
  delay(1000);
  anguloObjetivo = obtenerGrados(); // Fijamos el rumbo inicial
  Serial.println("Sensor listo.");
}
```
Este subprograma sirve para preparar el sensor antes de empezar a usar el robot. El programa espera hasta que el giroscopio esté bien calibrado. Mientras no esté bien calibrado, el robot no hace nada. Cuando ya está listo, espera un segundo y guarda la dirección actual como anguloObjetivo, es decir, el rumbo inicial que quiere mantener o usar como referencia. Luego avisa por pantalla que el sensor ya está listo.
```C++
void hacerGiro(int grados) {
  anguloObjetivo = anguloObjetivo + grados;

  // Ajuste para no salir del rango 0-360
  if (anguloObjetivo >= 360) anguloObjetivo -= 360;
  if (anguloObjetivo < 0) anguloObjetivo += 360;

  // Movemos el servo para girar
  if (grados > 0) cochino.write(centroServo + 35);
  else cochino.write(centroServo - 35);

  digitalWrite(IN1, LOW);
  digitalWrite(IN2, HIGH);
  analogWrite(ENA, 140);
```
Este subprograma sirve para girar el robot una cantidad exacta de grados. Primero calcula el nuevo objetivo sumando los grados que quiere girar. Después ajusta ese valor para que siempre esté entre 0 y 360 grados, como si fuera un círculo. Luego el robot empieza a girar: mueve el servo para ayudar al giro (dependiendo si gira a la derecha o izquierda), activa el motor en una dirección y le da una velocidad más lenta para que el giro sea más preciso y le de tiempo a girar sin problema. 
``` C++
 float dist;
  do {
    float actual = obtenerGrados();
    dist = anguloObjetivo - actual;
    if (dist > 180) dist -= 360;
    if (dist < -180) dist += 360;
  } while (abs(dist) > 5); // El bucle para cuando el error es menor a 5 grados

  cochino.write(centroServo); // Enderezamos ruedas
  delay(200);
```
Mientras el robot está girando, entra en un bucle donde está constantemente mirando su orientación actual con el sensor. Va comparando la posición actual con la posición objetivo y calcula el error (la diferencia entre ambos ángulos). Si esa diferencia es mayor de 5 grados, sigue girando. Cuando finalmente el robot está lo suficientemente cerca del ángulo deseado (menos de 5 grados de error), se detiene el giro, endereza las ruedas y espera un pequeño tiempo para estabilizarse.

``` C++
void terminarCarrera() {
  Serial.println("¡Última vuelta completada! Cruzando meta...");
  cochino.write(centroServo);
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, HIGH);
  analogWrite(ENA, 180);
  delay(2000); // Avanza 2 segundos extra para quedar bien posicionado al final
  
  analogWrite(ENA, 0); 
  digitalWrite(IN2, LOW);
  Serial.println("Carrera finalizada con éxito.");
}
```
Esta función sirve para que, cuando el robot termina la carrera, no intente girar ni corregir la dirección, sino que se mantenga completamente recto. Para eso primero centra el servo y luego hace que avance en línea recta durante unos segundos y entonces se detiene por completo. Este subprograma está pensado para asegurar que al final el robot siga recto y que no le pille el contador de giros a mitad de uno. 

# Explicación del programa - Obstacle Challenge
## Librerías
```C++
#include <Wire.h>
#include <SPI.h>
#include <Adafruit_Sensor.h>
#include <Adafruit_BNO055.h>
#include <Servo.h>
#include <Ultrasonic.h>
#include <Pixy2.h>
```
Al principio del código se incluyen varias librerías, que son conjuntos de funciones ya programadas por otros que nos permiten controlar componentes complejos sin tener que programar desde cero cómo funcionan. 
- `Wire` y `SPI` son protocolos de comunicación entre la placa y otros componentes.
- `Adafruit_BNO055` permite interactuar con el giroscopio.
-  `Servo` controla el servomotor que actúa como dirección.
- `Ultrasonic` gestiona los sensores de distancia por ultrasonidos
- `Pixy2` es la librería específica de la cámara inteligente que reconoce colores.

## Pines y Variables
```C++
int ENA = 5;
int IN1 = 6;
int IN2 = 7;
Servo cochino;
const int pinservo = 9;
const int pinPulsador = 13;

float anguloObjetivo;
int centroServo = 35;
int sentido = 0;
int contadorGiros = 0;
```
- A continuación se asignan los pines del Arduino a cada componente físico. `ENA`, ·`IN1` e `IN2` corresponden al motor de tracción, donde regulamos velocidad y sentido de giro de las ruedas.
- `pinservo` indica en qué pin está conectado el servomotor que dirige la dirección.
-  `pinPulsador` es el botón que el usuario pulsa para iniciar la carrera.

Después se declaran variables que actúan como la "memoria de trabajo" del robot durante su funcionamiento: `anguloObjetivo` almacena el ángulo hacia el que el robot debe orientarse para avanzar en línea recta; `centroServo` es la posición recta del servomotor; `sentido` determina si el circuito se recorre girando siempre hacia la derecha o hacia la izquierda; y `contadorGiros` lleva la cuenta de cuántas esquinas ha superado el robot, lo cual permite saber cuándo ha completado las tres vueltas.

## Cámara
```cpp
Adafruit_BNO055 bno = Adafruit_BNO055(55);
Pixy2 pixy;
#define SIG_ROJO  1
#define SIG_VERDE 2
const int GIRO_COLOR = 15;
```
Aquí se crean los objetos que representan al giroscopio (`bno`) y a la cámara (`pixy`). En programación orientada a objetos, esto equivale a decir que ya disponemos de esos dos instrumentos preparados para ser utilizados mediante sus respectivas funciones a lo largo del programa. La cámara Pixy2 no reconoce los colores por sí misma de forma innata: es necesario entrenarla previamente mediante un software externo llamado PixyMon, instalado en un ordenador. Durante ese entrenamiento se le muestra un objeto de un color determinado y se le asigna un número identificativo, denominado "signature". En este programa se asume que la signature número 1 corresponde al rojo y la signature número 2 al verde, aunque este orden depende exclusivamente de cómo se haya realizado el entrenamiento. La constante `GIRO_COLOR` determina cuántos grados se desviará el servomotor cuando se detecte uno de estos colores.

## Ultrasonidos
```cpp
#define TRIG_IZQ 8
#define ECHO_IZQ 10
Ultrasonic ultraIZQ(TRIG_IZQ, ECHO_IZQ, 60000);
```
El robot dispone de tres sensores de ultrasonidos —izquierdo, central y derecho— que funcionan de tal manera: emiten una onda sonora de alta frecuencia y miden el tiempo que tarda en regresar tras rebotar en un obstáculo. A partir de ese tiempo, y conociendo la velocidad del sonido en el aire, se calcula la distancia a la que se encuentra dicho obstáculo.

## Void Set up
```cpp
void setup() {
  Serial.begin(9600);
  ...
  if (!bno.begin()) { while(1); }
  pixy.init();
  prepararSensor();
}
```
La función `setup()` se ejecuta una única vez, justo al encender el Arduino, y su finalidad es preparar todos los componentes antes de que el robot empiece a moverse. En ella se inicializa la comunicación serie (útil para revisar las medidas desde el ordenador), se configuran los pines, se centra el servomotor, y se comprueba que el giroscopio responde correctamente; si no se detecta, el programa entra en un bucle infinito y detiene toda actividad, ya que sin ese sensor el robot no podría orientarse. A continuación se inicializa la cámara Pixy2 y, por último, se llama a la función `prepararSensor()`, que se explicará más adelante.

## Void Loop
```cpp
if (digitalRead(pinPulsador) == LOW) {
  start = true;
}
if (contadorGiros >= 12) {
  terminarCarrera();
  while(1);
}
int cmCentro = ultraCENTRO.Ranging(CM);
float anguloAhora = obtenerGrados();
int cmIzq = ultraIZQ.Ranging(CM);
int cmDer = ultraDERECH.Ranging(CM);

if (cmCentro < 80) { contador1++; } else { contador1 = 0; }
if (cmCentro < 140 && abs(cmIzq-cmDer) > 120) { contador2++; } else { contador2 = 0; }
if (contador1 > 2 || contador2 > 2) {
  // para el motor
  // si es la primera esquina, decide si el circuito es horario o antihorario
  // gira 90°
  contadorGiros++;
}
else {
  // avanza
  int colorDetectado = detectarColor();

  if (colorDetectado == SIG_VERDE) {
    cochino.write(centroServo - GIRO_COLOR);   // gira un poco a la izquierda
  } else if (colorDetectado == SIG_ROJO) {
    cochino.write(centroServo + GIRO_COLOR);   // gira un poco a la derecha
  } else {
    // sin colores: corrige el rumbo con el giroscopio, como antes
  }
}
```
La función `loop()` constituye el núcleo del programa, ya que se ejecuta de manera repetida y continua mientras el Arduino permanezca encendido, miles de veces por segundo.
Espera del pulsador. Al inicio del bucle se comprueba si se ha pulsado el botón de arranque. Mientras esto no ocurra, la variable `start` permanece en `false` y el resto del código no llega a ejecutarse de forma activa.
Condición de finalización. Una vez iniciada la carrera, se comprueba si `contadorGiros` ha alcanzado el valor 12, lo que equivale a tres vueltas completas, considerando que cada vuelta tiene cuatro esquinas. En ese caso, se llama a la función `terminarCarrera()` y el programa se detiene definitivamente mediante un bucle infinito vacío.
Lectura de sensores. En cada iteración se obtienen las distancias medidas por los tres sensores de ultrasonidos y el ángulo actual de orientación proporcionado por el giroscopio, mediante la función auxiliar `obtenerGrados()`.
Filtrado de lecturas. Para evitar que una lectura puntual errónea provoque un giro incorrecto, el programa no reacciona ante una sola detección de pared, sino que utiliza dos contadores (`contador1` y `contador2`) que se incrementan únicamente cuando la condición de proximidad se cumple varias veces consecutivas. Este mecanismo actúa como un filtro de seguridad frente al ruido característico de los sensores ultrasónicos.
Comportamiento ante un obstáculo. Cuando los contadores superan el umbral establecido, el motor se detiene, y si se trata de la primera esquina detectada en toda la carrera, el programa decide de manera autónoma si el circuito se recorrerá en sentido horario o antihorario, comparando qué lateral tiene mayor distancia libre. Seguidamente, se ejecuta la función `hacerGiro()` para realizar el giro de noventa grados correspondiente.
Comportamiento con el camino libre. Es en esta parte del programa donde se ha integrado la cámara. El robot avanza y, antes de aplicar la corrección habitual del rumbo, consulta a la función `detectarColor()` si se está detectando un bloque rojo o verde. Si el resultado es verde, el servomotor se desplaza ligeramente hacia la izquierda respecto a su posición central; si es rojo, se desplaza hacia la derecha. Únicamente cuando no se detecta ninguno de estos dos colores, el programa recurre a su lógica original: comparar el ángulo actual con el ángulo objetivo y corregir la dirección del servomotor en consecuencia, para mantener una trayectoria recta.


