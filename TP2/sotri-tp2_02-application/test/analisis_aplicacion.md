# Análisis y Explicación de la Aplicación (Sistemas de Tiempo Real con FreeRTOS)

Este documento analiza el funcionamiento del código de la aplicación de usuario implementada sobre **FreeRTOS** para la placa de desarrollo con el microcontrolador **STM32F446RE**. El firmware implementa un **sistema reactivo disparado por eventos** (Event-Triggered System, ETS) que gestiona la interacción entre un botón de usuario (B1) y un LED (LD2), filtrando rebotes por software y conmutando entre un estado de LED apagado y otro de parpadeo a 1 Hz.

---

## 1. Funcionamiento Detallado de los Archivos de la Aplicación

### A. `app.c`
Es el archivo coordinador de la inicialización de la capa de aplicación. Su función `app_init()` realiza lo siguiente:
1. **Inicialización de Variables de Control Global:** Pone a cero los contadores de la aplicación (`g_app_cnt`, `g_app_task_cnt`, `g_app_tick_cnt`, `g_task_idle_cnt`, `g_app_stack_overflow_cnt`) que permiten auditar el comportamiento del sistema operativo.
2. **Creación de Objetos de Comunicación Inter-tarea:**
   - **Cola (`h_btn_led_q`):** Se crea una cola con capacidad para 5 elementos del tipo `task_led_ev_t` (eventos de LED). Se registra su handle en el registro de depuración de FreeRTOS con el nombre `"BTN to LED Queue Handle"`.
   - **Semáforo Binario (`h_btn_led_bin_sem`):** Se crea un semáforo binario inicializado en el estado "vacío" (no tomado), registrándose como `"BTN to LED Binary Semaphore Handle"`.
3. **Creación de las Tareas de FreeRTOS:**
   - Crea la tarea del botón (**"Task BTN"**) con prioridad de ejecución de `1` (`tskIDLE_PRIORITY + 1ul`) y pila de `2 * configMINIMAL_STACK_SIZE` (256 words / 1024 bytes). El punto de entrada es la función `task_btn`.
   - Crea la tarea del LED (**"Task LED"**) con la misma prioridad de ejecución (`1`) e idéntico tamaño de pila. El punto de entrada es la función `task_led`.
4. **Llamadas de Inicialización Adicionales:**
   - Invoca a `app_it_init()` para preparar las interrupciones específicas de la aplicación.
   - Llama a `cycle_counter_init()` para habilitar el contador de ciclos del procesador (DWT) útil en mediciones precisas de ejecución.

### B. `app_it.c`
Se encarga de gestionar los callbacks de interrupción asociados a los periféricos de la capa de aplicación:
1. **`app_it_init()`:** Función vacía preparada para configuraciones de interrupción específicas del usuario. Ejecuta instrucciones en ensamblador para deshabilitar (`CPSID i`) y habilitar (`CPSIE i`) las interrupciones globales de forma segura.
2. **`HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)`:** Es la función de callback de la HAL de ST que se ejecuta cuando ocurre una interrupción por hardware en una línea EXTI (pines configurados con interrupción externa).
   - El código realiza una comprobación para determinar si la interrupción fue generada por el pin del botón `BTN_A_PIN`. Sin embargo, el bloque de código interno de esta condición está vacío (`/* Work to be done. */`). Esto significa que **las interrupciones físicas del botón son ignoradas por la lógica de la aplicación actual**.

### C. `task_btn.c`
Implementa la tarea encargada de leer el botón del usuario, resolver el anti-rebote (debouncing) por software y enviar señales a la tarea del LED.
1. **Instancia de Datos de Tarea (`task_btn_dta`):** Una estructura global privada que almacena el estado actual (`state`), el evento actual (`event`), la marca de tiempo del último cambio (`tick`), y la configuración física de puerto y pin del botón (B1 en PC13).
2. **Estructura de la Tarea (`task_btn`):**
   - Ejecuta un bucle infinito que incrementa el contador `g_task_btn_cnt`.
   - Invoca de forma iterativa a la máquina de estados `task_btn_statechart()`.
   - Llama a `vTaskDelay(BTN_TICK_DEL_MAX)` que suspende la tarea durante **50 ms** (`BTN_TICK_DEL_MAX` es 50 ms), permitiendo que otras tareas de igual o menor prioridad se ejecuten en ese intervalo.
3. **Máquina de Estados de Anti-rebote (`task_btn_statechart()`):**
   - Lee el estado del pin físico con `HAL_GPIO_ReadPin()`. Si está presionado, inyecta el evento interno `EV_BTN_DOWN`. En caso contrario, inyecta `EV_BTN_UP`.
   - Procesa las transiciones de estado para filtrar rebotes mecánicos mediante retardos temporales basados en ticks (detallado en la Sección 2).

### D. `task_led.c`
Implementa la tarea que controla el LED integrado (LD2 en PA5) basándose en las órdenes recibidas de la tarea del botón.
1. **Instancia de Datos de Tarea (`task_led_dta`):** Estructura que almacena si hay un evento pendiente (`flag`), el tipo de evento recibido (`event`), el estado actual (`state`), la marca de tiempo de la última acción (`tick`) y la configuración del puerto/pin del LED.
2. **Estructura de la Tarea (`task_led`):**
   - Inicializa el pin físico del LED apagándolo.
   - Utiliza una variable `last_wake_time` inicializada con `xTaskGetTickCount()`.
   - Ejecuta un bucle infinito que incrementa el contador `g_task_led_cnt` y ejecuta `task_led_statechart()`.
   - Aplica **`vTaskDelayUntil(&last_wake_time, LED_TICK_DEL_MAX)`**. Esto asegura una ejecución estrictamente periódica cada **50 ms**, compensando cualquier jitter o variación en el tiempo de procesamiento de la propia máquina de estados.
3. **Máquina de Estados del LED (`task_led_statechart()`):**
   - Implementa los estados `ST_LED_OFF` y `ST_LED_BLINK` (con un parpadeo periódico cada 500 ms). Responde de forma reactiva al cambio de variables globales y banderas (detallado en la Sección 2).

### E. `task_led_interface.c`
Define la interfaz de comunicación ("API") para que componentes externos (en este caso, la tarea del botón) envíen eventos a la tarea del LED:
- **`put_event_task_led(task_led_ev_t event)`:**
  - Escribe el evento directamente en la estructura compartida del LED: `task_led_dta.event = event;`
  - Establece la bandera de novedad en `true`: `task_led_dta.flag = true;`

### F. `freertos.c`
Contiene funciones Callback tipo Hook de FreeRTOS generadas por CubeMX:
- **`vApplicationIdleHook()`:** Se ejecuta en cada iteración de la tarea `Idle` (cuando ninguna tarea de usuario está lista). Permite poner al microcontrolador en modo de bajo consumo o auditar el tiempo libre del procesador.
- **`vApplicationTickHook()`:** Se ejecuta en contexto de interrupción de hardware con cada tick del sistema operativo (cada 1 ms). Útil para temporizaciones de alta precisión fuera del planificador.
- **`vApplicationStackOverflowHook(...)`:** Callback de protección. Si el kernel detecta que alguna tarea ha corrompido su límite de pila asignado, esta función se dispara, permitiendo registrar la falla o reiniciar de forma segura.

---

## 2. Modelado de las Máquinas de Estado (Statecharts)

La lógica de control de ambas tareas se basa en un diseño formal de máquinas de estado finitas tipo *Run-to-Completion*.

### A. Máquina de Estados del Botón (`task_btn_statechart`)
Su función primaria es filtrar el ruido eléctrico producido al presionar o liberar físicamente el botón (rebotes que suelen durar entre 5 y 20 ms) antes de confirmar el evento.

```
       +---------------------------------------------+
       |                                             | (Ruido o rebote en flanco)
       v                                             |
   [ST_BTN_UP] --------(EV_BTN_DOWN)--------> [ST_BTN_FALLING]
       ^                                             |
       |                                             | (Pasaron >= 50ms y sigue presionado)
       |                                             v
       |                                      [Establecer Tick]
       |                                      [Log: BTN PRESSED]
       |                                      [put_event_task_led(EV_LED_BLINK)]
       |                                             |
       |                                             v
       +------------(Pasaron >= 50ms)----------- [ST_BTN_DOWN]
       |             y sigue liberado                ^
       |                                             |
   [ST_BTN_RISING] <--------(EV_BTN_UP)--------------+
       ^                                             |
       |                                             | (Ruido o rebote en flanco)
       +---------------------------------------------+
```

#### Descripción de Estados y Transiciones:
- **`ST_BTN_UP` (Estado Estable: Botón Liberado):**
  - El pin físico lee nivel alto. Si el pin cambia a nivel bajo (`EV_BTN_DOWN`), se sospecha una pulsación: se registra el tick actual (`task_btn_dta.tick = xTaskGetTickCount()`) y se pasa a `ST_BTN_FALLING`.
- **`ST_BTN_FALLING` (Estado Transitorio: Filtrando Presión):**
  - Se espera a que transcurran al menos 50 ms (`DEL_BTN_MAX`). En la siguiente iteración de la tarea (transcurridos 50 ms):
    - Si el botón **sigue presionado** (`EV_BTN_DOWN`), se confirma la acción: se emite el log `"Task BTN - BTN PRESSED"`, se envía el evento de parpadeo a la tarea LED (`put_event_task_led(EV_LED_BLINK)`) y se pasa al estado estable `ST_BTN_DOWN`.
    - Si el botón se liberó (fue un rebote o ruido), se retorna inmediatamente a `ST_BTN_UP`.
- **`ST_BTN_DOWN` (Estado Estable: Botón Presionado):**
  - El pin lee nivel bajo. Si el pin cambia a nivel alto (`EV_BTN_UP`), se sospecha una liberación: se registra el tick actual y se transita a `ST_BTN_RISING`.
- **`ST_BTN_RISING` (Estado Transitorio: Filtrando Liberación):**
  - Se espera a que transcurran 50 ms. Pasado ese tiempo:
    - Si el botón **sigue liberado** (`EV_BTN_UP`), se confirma la liberación: se emite el log `"Task BTN - BTN HOVER"`, se envía el comando de apagado al LED (`put_event_task_led(EV_LED_OFF)`) y se retorna a `ST_BTN_UP`.
    - Si el botón volvió a presionarse, se retorna a `ST_BTN_DOWN`.

---

### B. Máquina de Estados del LED (`task_led_statechart`)
Controla el comportamiento visual del LED en base a los eventos inyectados a través de su interfaz.

```
   +-------------------------------------------------+
   |                                                 |
   v                                                 | (No hay evento OFF o pasaron < 500ms)
[ST_LED_OFF]                                         |
     |                                               v
     | (flag == true && event == EV_LED_BLINK)  [ST_LED_BLINK] --(Pasaron >= 500ms)--> [Toggle LED Pin]
     | [Log: LED BLINK]                              |
     | [Establecer Tick]                             | (flag == true && event == EV_LED_OFF)
     | [Encender LED]                                | [Log: LED OFF]
     v                                               v [Apagar LED]
     +-----------------------------------------------+
```

#### Descripción de Estados y Transiciones:
- **`ST_LED_OFF` (Estado Estable: LED Apagado):**
  - El LED físico se encuentra apagado.
  - Si llega una señal externa (`flag == true` y `event == EV_LED_BLINK`):
    - Se borra el flag (`flag = false`), se guarda el tick actual, se enciende físicamente el LED (`LED_ON`), se emite el log `"Task LED - LED BLINK"` y se transita a `ST_LED_BLINK`.
- **`ST_LED_BLINK` (Estado Estable/Dinámico: Parpadeo a 1 Hz):**
  - El sistema verifica continuamente si ha llegado una orden de apagado (`flag == true` y `event == EV_LED_OFF`):
    - De ser así, borra el flag, apaga el LED físico (`LED_OFF`), emite el log `"Task LED - LED OFF"` y regresa a `ST_LED_OFF`.
  - Si no hay evento de apagado, gestiona el parpadeo periódico:
    - Verifica si la diferencia entre el tick actual y el último guardado supera los **500 ms** (`DEL_LED_MAX`).
    - Al cumplirse los 500 ms, invoca a `HAL_GPIO_TogglePin()` para invertir el estado físico del LED y actualiza el tick de referencia (`tick = xTaskGetTickCount()`). Esto produce un ciclo de parpadeo simétrico de 1 Hz (500 ms encendido, 500 ms apagado).

---

## 3. Análisis Crítico del Diseño de la Aplicación

Al examinar integralmente el firmware, se evidencian dos discrepancias arquitectónicas sumamente importantes entre la configuración de hardware/periféricos del sistema y la implementación real del código de usuario.

### A. Discrepancia 1: Sondeo (Polling) vs. Interrupciones en el Botón

#### Lo que está configurado en el Hardware/HAL:
En `main.c` (dentro de `MX_GPIO_Init()`), el pin del botón (`B1_Pin`) se configura formalmente en modo de interrupción externa en flanco descendente (`GPIO_InitStruct.Mode = GPIO_MODE_IT_FALLING`). Adicionalmente, se habilita la línea de interrupción en el NVIC con prioridad 5 (`HAL_NVIC_EnableIRQ(EXTI15_10_IRQn)`). Esto significa que cada pulsación del botón genera una interrupción por hardware inmediata que desvía el flujo de ejecución hacia la ISR `EXTI15_10_IRQHandler` y posteriormente al callback `HAL_GPIO_EXTI_Callback()`.

#### Lo que hace el Código de Usuario:
- En `app_it.c`, el callback de interrupción de hardware del botón está completamente vacío (`HAL_GPIO_EXTI_Callback` no realiza ninguna acción).
- En su lugar, el código del botón se ejecuta de forma periódica dentro de la tarea `Task BTN` cada **50 ms** utilizando un esquema de **Sondeo Activo (Polling)** directo mediante la función `HAL_GPIO_ReadPin()`.

#### Impacto en el Diseño:
1. **Desperdicio de Recursos de CPU:** Al realizar Polling, la CPU se ve obligada a despertar a la tarea del botón cada 50 ms para leer el pin, consumiendo ciclos de procesamiento y energía de forma innecesaria cuando el botón no está siendo presionado (que suele ser el 99.9% del tiempo).
2. **Latencia de Respuesta:** Si el botón se presiona justo después de que la tarea entró en su delay de 50 ms, la aplicación tardará hasta 50 ms adicionales solo en enterarse de que el pin ha cambiado de estado, sumando latencia innecesaria.
3. **Solución Ideal (Event-Driven Real):** Se debería aprovechar la interrupción configurada (EXTI). Al ocurrir la presión física, la ISR en `app_it.c` debería capturar el evento y notificar de forma inmediata a la tarea de procesamiento del botón utilizando un semáforo o una cola, permitiendo que la tarea `Task BTN` permanezca en estado bloqueado de forma indefinida y consuma cero ciclos de CPU hasta que sea estrictamente necesario.

---

### B. Discrepancia 2: Mecanismo de Comunicación No Protegido (Banderas Globales) vs. Colas y Semáforos Desaprovechados

#### Lo que está Configurado y Creado:
En la función `app_init()`, se reserva memoria y se crean exitosamente dos potentes objetos de sincronización y comunicación de FreeRTOS:
1. Una **Cola de Mensajes (`h_btn_led_q`)** diseñada para almacenar hasta 5 eventos.
2. Un **Semáforo Binario (`h_btn_led_bin_sem`)** de sincronización.

#### Lo que hace el Código de Usuario:
- **Ninguno de estos objetos se utiliza en la aplicación.**
- En su lugar, la comunicación entre la tarea del botón (`Task BTN`) y la tarea del LED (`Task LED`) se implementa a través de la función `put_event_task_led()`, la cual escribe directamente sobre variables de estructura global no protegidas de la tarea LED (`task_led_dta.event` y `task_led_dta.flag`).

#### Impacto en el Diseño y Seguridad de Hilos (Thread-Safety):
1. **Condición de Carrera (Race Condition):** Aunque actualmente ambas tareas corren a la misma prioridad (`1`), FreeRTOS tiene activo el modo de desalojo (`configUSE_PREEMPTION = 1`) y el reparto de tiempo (*time-slicing*). Si se diera una interrupción en medio del acceso a la estructura global `task_led_dta` o si las tareas tuvieran diferentes prioridades, la modificación concurrente de variables globales compartidas sin mecanismos de exclusión mutua (como Mutexes o secciones críticas) provocaría corrupción de datos o comportamientos erráticos.
2. **Sondeo Ineficiente en la Tarea LED:** La tarea `Task LED` se ejecuta rígidamente cada 50 ms (`vTaskDelayUntil`) solo para verificar si la variable `flag` se ha puesto en `true`. Si no hay ningún cambio de estado del LED (el caso común), la tarea se despierta, consume CPU procesando las condicionales y vuelve a dormirse.
3. **Solución Ideal mediante Cola (`Queue`):**
   - **En `task_led_interface.c`:** Redefinir la API para enviar el evento directamente a la cola que ya está creada:
     ```c
     void put_event_task_led(task_led_ev_t event)
     {
         // Envía el evento a la cola. Bloquea un máximo de 0 ms si está llena.
         xQueueSend(h_btn_led_q, &event, 0);
     }
     ```
   - **En `task_led.c` (Bucle de la Tarea LED):** En lugar de despertar de forma periódica cada 50 ms a evaluar condicionales, la tarea LED debería bloquearse indefinidamente esperando recibir un dato de la cola:
     ```c
     void task_led(void *parameters)
     {
         task_led_ev_t received_event;
         for (;;)
         {
             // La tarea se bloquea completamente (0% de uso de CPU)
             // hasta que llegue un mensaje a la cola.
             if (pdPASS == xQueueReceive(h_btn_led_q, &received_event, portMAX_DELAY))
             {
                 // Al llegar un evento, se despierta instantáneamente
                 task_led_dta.event = received_event;
                 task_led_dta.flag = true;
                 task_led_statechart();
             }
         }
     }
     ```
#### Ventajas de la Solución Correcta (Usando la Cola Creada):
- **Eficiencia del CPU al Máximo (0% Overhead):** Las tareas consumen tiempo de procesamiento única y exclusivamente cuando ocurre un cambio físico en el botón, manteniéndose en estado bloqueado (suspendidas de forma pasiva por el planificador) el resto del tiempo.
- **Thread-Safety Garantizado:** La comunicación por cola en FreeRTOS es intrínsecamente segura frente a accesos concurrentes (maneja secciones críticas y colas de bloqueo internamente), protegiendo la integridad de los datos entre diferentes prioridades e interrupciones del procesador.
- **Acoplamiento Débil:** Se respeta la arquitectura modular de software donde los hilos no comparten memoria global de forma directa, sino a través de canales de paso de mensajes bien definidos.
