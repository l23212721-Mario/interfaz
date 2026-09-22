# Matriz de LEDs y técnica de barrido (_Charlieplexing_)

## 1. Introducción

En el diseño de sistemas embebidos y dispositivos con restricciones de área, la gestión eficiente de las líneas de Entrada/Salida de Propósito General (GPIO) representa un factor crítico. Tradicionalmente, el direccionamiento de una matriz de diodos emisores de luz (LED) se implementa mediante esquemas ortogonales multiplexados de filas y columnas en cuadrículas $M \times N$, lo cual demanda $M + N$ pines de control. 

Para superar este límite físico sin incorporar controladores dedicados o registros de desplazamiento, la técnica de multiplexación tri-state —formalizada comercialmente como *Charlieplexing* por Charlie Allen en 1995— permite controlar una cantidad sustancialmente mayor de actuadores empleando únicamente $N$ terminales. Este documento analiza la fundamentación física, la modelación matemática, los desafíos eléctricos de conmutación en bajo nivel y el diseño del algoritmo de barrido temporal necesario para su implementación.

---

## 2. Desarrollo Técnico

### 2.1 Principio de Operación y Lógica Tri-State (Hi-Z)

El pilar fundamental del Charlieplexing radica en la explotación del tercer estado lógico presente en las etapas de salida de los microcontroladores contemporáneos (CMOS push-pull):
1. **Nivel Lógico Alto (HIGH):** La terminal se conecta internamente a la fuente de voltaje mediante un transistor PMOS saturado.
2. **Nivel Lógico Bajo (LOW):** La terminal se conmuta a tierra mediante un transistor NMOS.
3. **Alta Impedancia (Hi-Z / Input):** Ambos transistores permanecen en corte. La impedancia de entrada se eleva al orden de los megaohmios, desacoplando efectivamente la terminal del circuito activo con corrientes de fuga insignificantes.

Al asociar pares de LEDs en oposición complementaria (antiparalelo) entre cada combinación posible de dos líneas de GPIO, solo circulará corriente a través del diodo polarizado en directa cuando un pin esté en nivel ALTO y el otro en BAJO. Los terminales restantes se configuran forzosamente en modo de entrada (Hi-Z), eliminando trayectorias no deseadas.

### 2.2 Modelado Combinatorio y Capacidad del Arreglo

Si se modela el circuito como un grafo dirigido donde cada terminal GPIO representa un nodo y cada LED representa un arco dirigido, el número máximo de elementos controlables independientemente $L$ con $N$ pines responde a la variación sin repetición de pares ordenados:

$$L = N(N - 1) = N^2 - N$$

A diferencia de una matriz convencional que con 8 pines (4 filas y 4 columnas) controla 16 LEDs, el Charlieplexing con $N = 8$ terminales es capaz de gobernar hasta 56 LEDs, demostrando una densidad de control que escala cuadráticamente.

### 2.3 Dinámica de Barrido Temporal y Persistencia Visual

Puesto que no es posible encender arbitrariamente dos diodos cualesquiera que compartan nodos sin inducir caminos parásitos, la matriz debe actualizarse mediante una secuencia acelerada aprovechando la Persistencia de la Visión (POV) de la retina humana.

Para garantizar una imagen libre de parpadeo perceptible (*flicker*), la frecuencia de cuadro global debe situarse en $f_{\text{frame}} \ge 60\text{ Hz}$. Para un banco de $L$ LEDs actualizados secuencialmente, la frecuencia de conmutación del temporizador se define por:

$$f_{\text{timer}} = L \cdot f_{\text{frame}}$$

Para $N=4$ líneas ($L=12$ LEDs) a 60 Hz, se requiere una interrupción de temporizador a $720\text{ Hz}$. El ciclo de trabajo efectivo (*duty cycle*) por diodo disminuye conforme se agregan pines. Para compensar la reducción en la emisión lumínica aparente, se debe incrementar la corriente de pulso instantánea, respetando los límites máximos por puerto del microcontrolador.

### 2.4 Comportamiento Eléctrico y Efecto Fantasma (*Ghosting*)

El principal desafío físico en estas matrices es el encendido involuntario de diodos adyacentes (*ghosting*), originado por dos causas:
1. **Disparidad de Tensión Directa ($V_F$):** Si conviven LEDs con distintas caídas de voltaje (ej. Azul de 3.2 V y Rojo de 1.8 V), un par de LEDs rojos en serie sobre ramales inactivos conectados a una línea flotante puede polarizarse en directa si su suma de tensiones es inferior al voltaje del riel.
2. **Capacitancias Parásitas y Tiempos de Conmutación:** Durante la transición entre estados lógicos en los registros, si un pin cambia de dirección con desfase frente al cambio de salida, se generan pulsos espurios. El firmware debe aplicar un "tiempo muerto" configurando todos los pines en entrada antes de activar el siguiente vector.

### 2.5 Implementación a Nivel de Registros

Para un control eficiente, la conmutación se programa manipulando directamente los registros de dirección (`DDR`) y puerto (`PORT`):

```c
#include <stdint.h>

typedef struct {
    uint8_t ddr_mask;   /* 1 = Salida, 0 = Entrada (Hi-Z) */
    uint8_t port_mask;  /* 1 = HIGH, 0 = LOW */
} CharlieStep;

static const CharlieStep charlie_lut[6] = {
    {0b00000011, 0b00000001}, /* LED 0 */
    {0b00000011, 0b00000010}, /* LED 1 */
    {0b00000110, 0b00000010}, /* LED 2 */
    {0b00000110, 0b00000100}, /* LED 3 */
    {0b00000101, 0b00000001}, /* LED 4 */
    {0b00000101, 0b00000100}  /* LED 5 */
};

void Timer_ISR_Handler(void) {
    static uint8_t current_led = 0;
    static uint8_t display_buffer = 0b00101101; 

    /* 1. Aislar pines a Hi-Z para evitar ghosting */
    GPIO_DDR_REG &= ~0b00000111; 
    GPIO_PORT_REG &= ~0b00000111;

    /* 2. Activar el LED actual si corresponde */
    if (display_buffer & (1 << current_led)) {
        GPIO_PORT_REG |= charlie_lut[current_led].port_mask;
        GPIO_DDR_REG  |= charlie_lut[current_led].ddr_mask;
    }

    if (++current_led >= 6) current_led = 0;
}
```

## 3. Conclusiones

La técnica de Charlieplexing representa una solución ingeniosa cuando la minimización de componentes y el conteo de terminales dictan la arquitectura del sistema. Demuestra cómo el conocimiento del diseño de las etapas digitales (tercer estado Hi-Z) soluciona restricciones de hardware mediante software.

No obstante, su viabilidad está condicionada por un compromiso técnico: a medida que el número de LEDs crece, el ciclo de trabajo decae, lo que impone límites estrictos en el brillo y demanda mayores picos de corriente. Por consiguiente, Charlieplexing es ideal para matrices pequeñas ($N \le 6$), mientras que matrices masivas dependen de arquitecturas convencionales.

## 4. Referencias Bibliográficas

[1] C. Allen, "Multiplexing scheme for driving multiple light-emitting diodes," U.S. Patent 5 869 986, Feb. 9, 1999.

[2] J. Morton, _AVR: The Essence of Microcontrollers_, 2nd ed. Oxford, UK: Newnes, 2004, pp. 112–118.

[3] Microchip Technology, "AVR284: Crossplexing or Charlieplexing with tinyAVR," Application Note AN284, 2013.

[4] Texas Instruments, "Techniques for driving LED displays with minimal GPIOs," Application Report SLAA789, Dallas, TX, USA, 2018.
