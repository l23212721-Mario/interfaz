## 1. Registro de Interacciones y Prompts Reales

### Sesión 1: Exploración teórica y modelado matemático
* **Prompt ingresado:**  
  > "Actúa como un ingeniero en sistemas embebidos y arquitectura de computadoras. Proporciona un desglose técnico riguroso sobre la técnica de Charlieplexing aplicada a matrices de diodos emisores de luz. Explica la deducción matemática de la fórmula N(N-1), el aprovechamiento del estado de alta impedancia (Hi-Z) de los pines tri-state de GPIO, el cálculo del ciclo de trabajo (duty cycle) necesario para evitar parpadeo perceptible (flicker) y el fenómeno de tensión inversa parásita (ghosting). Excluye introducciones genéricas y enfócate en el comportamiento eléctrico a bajo nivel."
* **Resultados obtenidos**  
  El modelo generó la deducción del análisis combinatorio de permutaciones, el diagrama de conexiones en antiparalelo y una explicación detallada del estado de alta impedancia (Hi-Z). Detalló con precisión la causa del *ghosting* cuando se combinan LEDs con diferentes caídas de voltaje directo (ej. diodos rojos frente a azules).

### Sesión 2: Diseño de código y manipulación de registros a bajo nivel
* **Prompt ingresado:**  
  > "Diseña una rutina en C y bajo nivel para controlar una cuadrícula de Charlieplexing de N=4 líneas. Muestra cómo se manipulan los registros de dirección de datos (DDR/MODER) y salida de datos (PORT/ODR) para conmutar un pin a nivel HIGH, otro a LOW y mantener los restantes en entrada/alta impedancia. Incluye el cálculo temporal del refresco a 60 Hz mediante temporizadores por hardware y los riesgos de sobrecorriente por pin."
* **Resultados obtenidos:**  
  El modelo entregó un fragmento de código estructurado con una tabla de búsqueda (LUT) de máscaras de bits y una rutina de interrupción (ISR). Propuso el uso de operadores a nivel de bits para limpiar y establecer los registros `DDR` y `PORT`.

---

## 2. Reflexión Crítica

* **¿Ayudó el LLM?**  
  Sí, el asistente funcionó como un excelente catalizador para estructurar el documento y aterrizar conceptos abstractos (como la persistencia de la visión) en fórmulas matemáticas de frecuencia concretas. Aceleró significativamente la redacción del marco teórico y proporcionó una base sintáctica sólida para el diseño de la lógica en C.

* **¿Hubo sesgos o errores detectados?**  
  Durante el análisis de la rutina de código generada por la IA, detecté un riesgo físico de *ghosting* y destellos espurios (glitches) derivado de la secuencia de conmutación. El modelo originalmente sugería cambiar el estado lógico de `PORT` un ciclo de reloj antes de aislar las líneas con el registro `DDR`. Al revisar el flujo y simular la lógica localmente en mi entorno de desarrollo en mi equipo Lenovo i7, tuve que corregir la rutina manualmente para forzar primero todos los pines a entrada (Hi-Z), garantizando un "tiempo muerto" antes de aplicar el nuevo vector de salida. Adicionalmente, el LLM omitió enfatizar los límites de disipación de corriente total del puerto del microcontrolador, un parámetro eléctrico crítico que tuve que investigar y añadir al reporte basándome en las hojas de datos reales de los fabricantes.
