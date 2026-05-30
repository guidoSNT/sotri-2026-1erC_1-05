# Análisis y Explicación del Código Fuente (STM32F446xx y FreeRTOS)

> **Nota Aclaratoria Inicial:** La consigna original menciona los archivos `startup_stm32f103rbtx.s` y `stm32f1xx_it.c`. Sin embargo, los archivos adjuntos provistos en la carpeta de pruebas (`test`) corresponden a un microcontrolador **STM32F446xx** (`startup_stm32f446retx.s` y `stm32f4xx_it.c`). Por lo tanto, este análisis se basa estrictamente en los archivos reales adjuntos, detallando el funcionamiento para el procesador **ARM Cortex-M4 (STM32F446RE)**. Los conceptos de inicialización, arquitectura de la HAL de ST y funcionamiento del kernel de FreeRTOS descritos aquí son plenamente análogos y equivalentes a los del STM32F103 (Cortex-M3).

---

## 1. Análisis y Funcionamiento de los Archivos

### A. `startup_stm32f446retx.s`
Es el archivo de arranque (escrito en lenguaje ensamblador ARM Cortex-M4). Sus funciones principales son:
1. **Definición de la Tabla de Vectores (`g_pfnVectors`):** Reserva el espacio de memoria inicial (ubicado típicamente en la dirección flash `0x08000000`) para colocar la dirección del tope de la pila (`_estack`) en la posición 0, seguida de la dirección de `Reset_Handler` en la posición 1, y luego las direcciones de todos los manejadores de excepciones del núcleo ARM (NMI, HardFault, SVC, PendSV, SysTick) y las interrupciones externas (periféricos específicos del microcontrolador, como USART2, TIM1, TIM2, etc.).
2. **Inicialización de la Pila (Stack):** Carga la dirección del puntero de pila (`sp`) con el símbolo del enlazador `_estack` (línea 61).
3. **Inicialización del Sistema de Relojes (`SystemInit`):** Realiza una llamada inicial mediante una bifurcación con enlace (`bl SystemInit`) para poner el oscilador interno (HSI) y los registros de configuración del reloj del sistema en su estado por defecto antes de inicializar las variables.
4. **Copia de la Sección de Datos (`.data`):** Copia los valores iniciales de las variables globales y estáticas inicializadas desde la memoria Flash (origen `_sidata`) a la memoria RAM (destino `_sdata` a `_edata`).
5. **Limpieza de la Sección BSS (`.bss`):** Inicializa con ceros (`0x00`) toda la región de memoria RAM dedicada a variables globales y estáticas no inicializadas (delimitada entre `_sbss` y `_ebss`).
6. **Llamada a Constructores Estáticos:** Llama a `__libc_init_array` para ejecutar inicializaciones de la biblioteca C estándar.
7. **Bifurcación al Punto de Entrada Principal (`main`):** Llama a la función `main` en C (`bl main`). Si esta llegase a retornar, entra en un bucle infinito (`bx lr`).

### B. `main.c`
Es el archivo principal de la aplicación. Contiene el punto de entrada `main()` y se encarga de:
1. **Inicialización de la HAL (`HAL_Init`):** Configura la latencia de la memoria flash, los cachés de instrucciones/datos y el temporizador que servirá de base de tiempo para la HAL de ST. **Crucialmente**, este proyecto está configurado para usar **TIM1** en lugar de SysTick como su base de tiempo de ticks de la HAL, evitando conflictos con FreeRTOS.
2. **Configuración de los Relojes (`SystemClock_Config`):** Configura el microcontrolador para trabajar a su velocidad objetivo utilizando el oscilador interno HSI de 16 MHz y el lazo de seguimiento de fase (PLL). 
   - Multiplica y divide el reloj para que el reloj del procesador (SYSCLK y HCLK) sea de **84 MHz**.
   - Configura los divisores de los buses periféricos APB1 (dividido por 2, dando un reloj de periféricos de 42 MHz y de temporizadores de 84 MHz) y APB2 (sin dividir, dando 84 MHz).
3. **Inicialización de Periféricos:**
   - `MX_GPIO_Init()`: Inicializa los pines GPIO, configurando el LED integrado (LD2 en PA5) como salida y habilitando la interrupción externa para el botón azul (B1 en PC13) en flanco descendente (prioridad 5).
   - `MX_USART2_UART_Init()`: Configura el puerto serie USART2 a 115200 baudios para depuración/logs.
   - `MX_TIM2_Init()`: Configura el temporizador **TIM2** con un prescaler de `2-1` y un período de `4200-1`. Esto genera interrupciones periódicas a una frecuencia de **10 kHz** (cada 100 µs), utilizada exclusivamente como contador de alta frecuencia para las estadísticas de uso de CPU de las tareas en FreeRTOS (`configGENERATE_RUN_TIME_STATS`).
4. **Inicio del Temporizador de Estadísticas:** Llama a `HAL_TIM_Base_Start_IT(&htim2)` para comenzar a contar con TIM2 bajo interrupciones.
5. **Inicialización de la Aplicación e hilos de FreeRTOS:**
   - Llama a `app_init()` para preparar las tareas de la aplicación.
   - Define y crea la tarea por defecto (`defaultTask` mediante `osThreadCreate`).
6. **Arranque del Kernel de FreeRTOS (`osKernelStart`):** Llama a `osKernelStart()` (que invoca a `vTaskStartScheduler()`), transfiriendo el control completo del procesador al planificador de tareas. El bucle `while (1)` al final de `main()` queda como código inalcanzable.

### C. `stm32f4xx_it.c`
Este archivo contiene las rutinas de servicio de interrupción (ISR) que gestionan las excepciones del procesador y los periféricos de hardware:
1. **Excepciones del Sistema (Cortex-M4):** Maneja las excepciones críticas del sistema como `NMI_Handler`, `HardFault_Handler`, `MemManage_Handler`, `BusFault_Handler`, y `UsageFault_Handler`. Todas estas, en caso de ocurrir, bloquean la ejecución en un bucle infinito `while(1)` para facilitar la depuración mediante JTAG/SWD.
2. **`TIM1_UP_TIM10_IRQHandler`:** Maneja la interrupción del temporizador **TIM1** (asociado a la base de tiempo de la HAL de ST). Invoca a `HAL_TIM_IRQHandler(&htim1)`, la cual a su vez llama a la función de callback `HAL_TIM_PeriodElapsedCallback()` para incrementar el contador de ticks de la HAL (`uwTick`).
3. **`TIM2_IRQHandler`:** Maneja la interrupción global del temporizador **TIM2**. Invoca a `HAL_TIM_IRQHandler(&htim2)`, la cual procesa la interrupción de TIM2 y llama a `HAL_TIM_PeriodElapsedCallback()`, donde se incrementa el contador de estadísticas de ejecución `ulHighFrequencyTimerTicks` de FreeRTOS.
4. **`EXTI15_10_IRQHandler`:** Maneja la interrupción de los pines GPIO 10 a 15 (en este caso, correspondiente al botón B1 en PC13). Invoca a `HAL_GPIO_EXTI_IRQHandler(B1_Pin)` para borrar la bandera de interrupción de línea y procesar el evento.

### D. `FreeRTOSConfig.h`
Es el archivo de cabecera de configuración del sistema operativo. Determina el comportamiento del planificador de FreeRTOS:
- `configUSE_PREEMPTION = 1`: Habilita el esquema de planificación por desalojo (preemption).
- `configSUPPORT_STATIC_ALLOCATION = 1`: Habilita la creación de tareas, colas y semáforos en memoria asignada estáticamente.
- `configSUPPORT_DYNAMIC_ALLOCATION = 1`: Habilita la asignación dinámica de memoria mediante el heap de FreeRTOS.
- `configCPU_CLOCK_HZ = ( SystemCoreClock )`: Informa al kernel de la frecuencia de funcionamiento de la CPU actual (obtenida dinámicamente mediante la variable CMSIS `SystemCoreClock`, que vale 84 MHz tras la inicialización).
- `configTICK_RATE_HZ = 1000`: Define el tick del sistema operativo en **1 ms** (1000 Hz).
- `configMAX_PRIORITIES = 7`: Define 7 niveles de prioridad para las tareas.
- `configGENERATE_RUN_TIME_STATS = 1`: Activa el subsistema para rastrear el tiempo de CPU consumido por cada tarea. Configura los macros `portCONFIGURE_TIMER_FOR_RUN_TIME_STATS` y `portGET_RUN_TIME_COUNTER_VALUE` mapeándolos a funciones definidas en `main.c`.
- **Definición de vectores de FreeRTOS:** Asocia las rutinas de interrupción de FreeRTOS con los nombres estándar de CMSIS:
  - `vPortSVCHandler` se mapea a `SVC_Handler`.
  - `xPortPendSVHandler` se mapea a `PendSV_Handler`.
  - `xPortSysTickHandler` se mapea a `SysTick_Handler` (reemplazando el manejador SysTick por defecto del sistema).

### E. `freertos.c`
Contiene la lógica de integración de FreeRTOS generada por el CubeMX:
1. **`vApplicationGetIdleTaskMemory()`:** Provee el buffer de control de tarea (`StaticTask_t`) y el array de pila estática requeridos para crear la tarea de Idle (obligatorio cuando `configSUPPORT_STATIC_ALLOCATION == 1`).
2. **Callbacks Auxiliares (Hooks débiles):** Define funciones declaradas como `__weak` para que la aplicación las sobrescriba si es necesario:
   - `vApplicationIdleHook()`: Código que se ejecuta continuamente cuando el sistema no tiene tareas listas y entra en Idle.
   - `vApplicationTickHook()`: Código ejecutado dentro de cada tick del sistema operativo (en contexto de ISR).
   - `vApplicationStackOverflowHook()`: Manejador de error en caso de que una tarea exceda su tamaño de pila.

---

## 2. Evolución de las Variables `SysTick` y `SystemCoreClock`

### A. Variable `SystemCoreClock` (Variable global CMSIS de frecuencia de CPU)
1. **Inicio en `Reset_Handler` (`startup_stm32f446retx.s`):**
   - Cuando el procesador arranca, lo hace utilizando el oscilador interno por defecto HSI. En el STM32F446xx, la frecuencia del HSI es de **16 MHz**.
   - Al ejecutarse `SystemInit`, la variable `SystemCoreClock` se inicializa con el valor por defecto de **`16000000`** (16 MHz).
2. **Durante `HAL_Init()` en `main.c`:**
   - Mantiene su valor de **`16000000`**, ya que los relojes del sistema aún no han sido modificados.
3. **Durante `SystemClock_Config()` en `main.c`:**
   - La función configura el oscilador y el PLL para lograr un reloj de CPU de **84 MHz**.
   - Al final de la configuración, el driver invoca a la función de CMSIS `SystemCoreClockUpdate()`. Esta función lee los registros de control de reloj del hardware (`RCC->CFGR` y `RCC->PLLCFGR`) y actualiza la variable en software.
   - En este punto, **`SystemCoreClock` pasa a valer `84000000`** (84 MHz).
4. **Hasta el `while (1)` de `main.c`:**
   - Permanece constante en **`84000000`**.

### B. Periférico `SysTick` y Variables de Ticks (`uwTick` de la HAL y `xTickCount` de FreeRTOS)
En esta configuración de software, el comportamiento de las bases de tiempo cambia drásticamente debido a la coexistencia de la HAL de ST y FreeRTOS:

1. **Inicio en `Reset_Handler` (`startup_stm32f446retx.s`):**
   - El periférico hardware `SysTick` está **completamente desactivado** (`SysTick->CTRL = 0`).
   - Las variables globales de ticks `uwTick` (HAL) y `xTickCount` (FreeRTOS) residen en la región `.bss` y son puestas a **`0`** por el código de ensamblaje en el startup.
2. **Durante `HAL_Init()` en `main.c`:**
   - El framework de ST llama internamente a `HAL_InitTick()`. Dado que se ha configurado **TIM1** como la base de tiempo de la HAL (para no colisionar con FreeRTOS), **TIM1 es configurado para interrumpir cada 1 ms** bajo la frecuencia actual (HSI = 16 MHz).
   - **`SysTick` sigue completamente desactivado.**
   - La variable de la HAL **`uwTick` comienza a incrementarse** en `1` cada milisegundo por la acción de la ISR de TIM1. La variable `xTickCount` de FreeRTOS sigue en `0`.
3. **Durante `SystemClock_Config()` en `main.c`:**
   - Tras configurar el reloj a 84 MHz, se vuelve a invocar a `HAL_InitTick()`. Esto reajusta los registros de carga de **TIM1** para mantener el tick de 1 ms bajo la nueva velocidad de 84 MHz.
   - **`SysTick` sigue desactivado.**
   - `uwTick` sigue incrementándose a ritmo constante de 1 kHz.
4. **Durante Inicialización de Periféricos y Creación de Tareas (`app_init()`, `osThreadCreate()`):**
   - No hay cambios en el estado del hardware de SysTick. `SysTick` sigue inactivo.
   - `uwTick` sigue incrementándose. `xTickCount` de FreeRTOS sigue valiendo `0`.
5. **Durante `osKernelStart()` / `vTaskStartScheduler()` (Arranque del Planificador):**
   - Las interrupciones globales se deshabilitan momentáneamente.
   - Se llama a la función interna del port de FreeRTOS `xPortStartScheduler()`, la cual realiza la **configuración y arranque del periférico SysTick**:
     - Carga el registro de recarga: `SysTick->LOAD = (84000000 / 1000) - 1 = 83999` (para una frecuencia de interrupción de 1000 Hz, es decir, 1 ms).
     - Limpia el registro de valor actual: `SysTick->VAL = 0`.
     - Configura el registro de control: `SysTick->CTRL` se establece con el reloj del procesador (sin división), habilitación de interrupción y habilitación del contador (`SysTick_CTRL_CLKSOURCE_Msk | SysTick_CTRL_TICKINT_Msk | SysTick_CTRL_ENABLE_Msk`).
     - Configura el registro del sistema de prioridades de interrupción para que SysTick tenga la prioridad más baja admisible (`15` en un esquema de 4 bits de prioridad).
   - Se habilitan las interrupciones globales mediante el arranque de la primera tarea (`SVC_Handler`).
6. **Ejecución de FreeRTOS (Antes del `while(1)` inalcanzable de `main.c`):**
   - El temporizador hardware **SysTick está activo e interrumpe cada 1 ms**. Cada interrupción ejecuta `SysTick_Handler` (mapeado a `xPortSysTickHandler` de FreeRTOS) que incrementa progresivamente la variable interna del kernel **`xTickCount`** (`xTickCount++`).
   - El temporizador **TIM1 sigue funcionando de forma independiente e interrumpe cada 1 ms**, llamando a `HAL_IncTick()` e incrementando de forma paralela la variable **`uwTick`** de la HAL.
   - Ambas variables de tick (`xTickCount` y `uwTick`) evolucionan concurrentemente sumando 1 tick por cada milisegundo que transcurre.

---

## 3. Comportamiento del Programa desde su Inicio hasta el Loop Principal

El flujo de control detallado del programa sigue la siguiente secuencia temporal y lógica desde el Reset físico del hardware:

1. **Reset Físico:** El procesador carga el puntero de pila (`SP`) con el valor de la dirección `0x08000000` y el program counter (`PC`) con la dirección del manejador de reset (`Reset_Handler`) en la dirección `0x08000004`.
2. **`Reset_Handler` (Ensamblador):**
   - Inicializa el puntero de pila principal en los registros del CPU.
   - Llama a `SystemInit()` para asegurar que el procesador arranque en un estado limpio con el oscilador interno (HSI a 16 MHz).
   - Copia las variables globales inicializadas de Flash a RAM.
   - Limpia (llena con ceros) el bloque de memoria de variables no inicializadas (`.bss`).
   - Llama a `__libc_init_array()`.
   - Realiza un salto (`bl main`) hacia el programa principal escrito en C.
3. **Inicio de `main()` (C):**
   - Si se definió el uso de semihosting, se llama a `initialise_monitor_handles()` para enlazar los descriptores de consola con la herramienta de depuración en la PC.
   - Ejecuta `HAL_Init()`: 
     - Configura la latencia de lectura de Flash.
     - Divide el NVIC en un grupo de prioridad de 4 bits para preemption y 0 bits para subprioridades.
     - Inicializa el temporizador **TIM1** para que sirva de reloj base de 1 ms de la HAL (incrementa la variable `uwTick`).
   - Ejecuta `SystemClock_Config()`:
     - Habilita el oscilador interno HSI.
     - Enciende el lazo PLL configurándolo para entregar un SYSCLK de **84 MHz**.
     - Selecciona la salida del PLL como el reloj maestro del procesador.
     - Actualiza la variable global `SystemCoreClock` a `84000000`.
     - Reconfigura el temporizador de la HAL (**TIM1**) para adaptarlo a la nueva frecuencia de 84 MHz y mantener su precisión de 1 ms.
   - Ejecuta las inicializaciones de los módulos de hardware:
     - `MX_GPIO_Init()`: Enciende los relojes de puerto GPIO, configura el LED de la placa y configura la línea EXTI para detectar presiones en el botón azul (B1).
     - `MX_USART2_UART_Init()`: Abre el canal de comunicación serie para el logger.
     - `MX_TIM2_Init()`: Inicializa el temporizador **TIM2** para que interrumpa cada 100 µs (10 kHz) para el muestreo de estadísticas de tareas.
   - Arranca el temporizador de estadísticas:
     - Llama a `HAL_TIM_Base_Start_IT(&htim2)` para habilitar las interrupciones de TIM2. A partir de este momento, TIM2 interrumpe periódicamente e incrementa la variable `ulHighFrequencyTimerTicks` (funcionando en segundo plano mediante la ISR `TIM2_IRQHandler`).
   - Inicializa la estructura del proyecto e hilos de FreeRTOS:
     - Llama a `app_init()` para que configure la lógica interna y las colas de la aplicación.
     - Crea la tarea por defecto (`defaultTask`) reservando estáticamente sus bloques de control (`TCB`) y espacio en pila, asignándole prioridad normal.
   - Arranca el Scheduler del Kernel de FreeRTOS (`osKernelStart()` / `vTaskStartScheduler()`):
     - El kernel reserva la memoria para la tarea del Idle (ejecutando internamente `vApplicationGetIdleTaskMemory`).
     - Deshabilita temporalmente las interrupciones para evitar que ocurran cambios de contexto durante el setup crítico del kernel.
     - Configura el periférico hardware **SysTick** para que empiece a decrementar desde un intervalo equivalente a **1 ms** y active su interrupción periódica de baja prioridad (prioridad 15).
     - Configura la prioridad de la interrupción `PendSV` (usada para el cambio de contexto) y `SVCall` (usada para lanzar la primera tarea) al nivel más bajo (15).
     - Restablece el valor de registro de la pila principal (MSP) para liberar toda la memoria que utilizó la pila durante el flujo de inicialización del `main()`. Esto evita el desperdicio de memoria RAM.
     - Ejecuta la instrucción de ensamblador `SVC 0` (SuperVisor Call) para disparar de forma controlada la excepción de llamada al sistema.
     - El procesador ejecuta `vPortSVCHandler()` (SVC_Handler), el cual selecciona la tarea de mayor prioridad lista para ejecutarse (en este punto es `defaultTask` o alguna tarea inicializada en `app_init()`), carga su contexto de registros y salta hacia su función de inicio.
     - Las interrupciones del CPU se vuelven a habilitar.
     - **El control es asumido en un 100% por FreeRTOS.** Las interrupciones de SysTick (cada 1 ms), TIM1 (cada 1 ms para la HAL) y TIM2 (cada 100 µs para estadísticas) se ejecutan de manera concurrente en segundo plano. El planificador ejecuta las tareas asignándoles tiempo de procesamiento según sus prioridades.
     - El bucle `while (1)` final de `main()` **nunca se llega a ejecutar**, quedando la línea de ejecución suspendida indefinidamente bajo el control de las tareas del sistema operativo de tiempo real.

---

## 4. Interacción de SysTick y el Timer de Estadísticas (TIM2) con FreeRTOS

Tanto el periférico `SysTick` como el Timer 2 (`TIM2`) juegan roles fundamentales en el ecosistema del sistema operativo de tiempo real, pero cumplen propósitos completamente distintos:

### A. SysTick e Interacción con FreeRTOS
El temporizador `SysTick` es el motor interno de FreeRTOS (denominado comúnmente como el **"Kernel Tick Source"**).

- **Cómo interactúa:**
  - El port de FreeRTOS configura este periférico (integrado directamente en el núcleo Cortex-M) durante la inicialización del planificador (`xPortStartScheduler`).
  - Se define la macro de compilación `#define xPortSysTickHandler SysTick_Handler` en `FreeRTOSConfig.h`. Esto hace que el compilador reemplace directamente la función de la tabla de vectores del procesador para que apunte a la rutina de interrupción de FreeRTOS.
  - El periférico genera una interrupción periódica cada **1 ms** (frecuencia de 1000 Hz, calculada como `SystemCoreClock / 1000`). Su prioridad se configura en la más baja posible (15) para que no bloquee ni agregue latencia ("jitter") a las interrupciones críticas de periféricos de hardware en tiempo real.
- **Para qué sirve:**
  - **Medición del Tiempo del Kernel:** En cada interrupción, se incrementa la variable interna de control de ticks de FreeRTOS (`xTickCount`).
  - **Gestión de Bloqueos por Tiempo (Delays/Timeouts):** El planificador evalúa si alguna de las tareas que estaban en estado bloqueado esperando que transcurriera un tiempo (por ejemplo, tareas en `vTaskDelay` o bloqueadas en colas/semáforos con un timeout de espera) ha cumplido su plazo. De ser así, las mueve al estado de listas para ejecutarse.
  - **Planificación por Tiempo Compartido (Time-slicing):** Si hay varias tareas ejecutándose con la misma prioridad y la apropiación está activa (`configUSE_PREEMPTION == 1`), cada tick de SysTick le permite al planificador quitarle de forma ordenada la CPU a la tarea actual y concedérsela a la siguiente tarea lista del mismo nivel, asegurando equidad en el reparto de tiempo de procesamiento.

### B. Timer 2 (TIM2) e Interacción con FreeRTOS
El temporizador `TIM2` se utiliza para el subsistema de análisis y monitoreo avanzado de FreeRTOS (denominado **"Run-Time Stats"**).

- **Cómo interactúa:**
  - En `FreeRTOSConfig.h`, se habilitan las estadísticas configurando la macro `configGENERATE_RUN_TIME_STATS` a `1`. 
  - Se mapea la inicialización de este temporizador a través de `#define portCONFIGURE_TIMER_FOR_RUN_TIME_STATS configureTimerForRunTimeStats` (función en `main.c` que pone a cero el contador `ulHighFrequencyTimerTicks`).
  - Se mapea la lectura del valor de tiempo mediante `#define portGET_RUN_TIME_COUNTER_VALUE getRunTimeCounterValue` (función en `main.c` que retorna el valor actual de `ulHighFrequencyTimerTicks`).
  - El hardware de **TIM2 se configura para interrumpir a una velocidad 10 veces mayor que el kernel tick (frecuencia de 10 kHz)**. Cada vez que TIM2 desborda, la ISR correspondiente (`TIM2_IRQHandler`) y el sistema de interrupciones de la HAL de ST incrementan el valor de la variable volátil `ulHighFrequencyTimerTicks`.
- **Para qué sirve:**
  - **Perfilado de CPU de Tareas (Profiling):** Permite a FreeRTOS llevar un control sumamente preciso de la cantidad de tiempo real que el procesador ha pasado ejecutando cada tarea individual (a través de la API `vTaskGetRunTimeStats`).
  - **Resolución Temporal:** Dado que el tick normal del kernel ocurre cada 1 ms, si usáramos la variable `xTickCount` para medir el uso de CPU de las tareas, el error de redondeo sería enorme (una tarea que tarda 0.2 ms podría registrarse como que consumió 0 ms o 1 ms completo). Al disponer de TIM2 corriendo a **10 kHz (muestreo cada 100 microsegundos)**, FreeRTOS obtiene la precisión necesaria para realizar mediciones realistas del rendimiento, permitiendo a los ingenieros detectar cuellos de botella en la aplicación.

---

## 5. Interacción del Timer 2 (TIM2) con la HAL de STM32

El temporizador `TIM2` es un periférico de hardware del microcontrolador y su control lógico y físico se implementa a través de la capa de abstracción de hardware (**STM32 HAL**). Su interacción sigue los siguientes patrones de diseño y flujo de llamadas:

### A. Configuración y Control del Ciclo de Vida del Periférico
La HAL proporciona las estructuras de datos y funciones para parametrizar e iniciar el hardware sin escribir directamente en los registros del microcontrolador:
1. **Representación de Datos:** La HAL encapsula el estado y registro del temporizador en un objeto ("handle") global denominado `htim2` de tipo `TIM_HandleTypeDef` (definido en `main.c`, línea 50).
2. **Inicialización Física (`MX_TIM2_Init`):** 
   - Configura la instancia de hardware asociada (`htim2.Instance = TIM2`).
   - Define el pre-escalador (`Prescaler = 2 - 1`) y el registro de auto-recarga (`Period = 4200 - 1`).
   - Llama a `HAL_TIM_Base_Init(&htim2)` para escribir estos parámetros de forma segura en los registros físicos de control del temporizador de STM32.
   - Configura la fuente del reloj interna de TIM2 mediante la llamada a `HAL_TIM_ConfigClockSource()`.
3. **Arranque en Modo Interrupción:** En `main.c` (línea 115), la aplicación invoca a `HAL_TIM_Base_Start_IT(&htim2)`. Esta función de la HAL realiza dos tareas críticas:
   - Habilita la interrupción por desbordamiento de TIM2 a nivel de registros periféricos (escribiendo en el bit `UIE` del registro `TIM2->DIER`).
   - Arranca el contador del temporizador (escribiendo en el bit `CEN` del registro `TIM2->CR1`).

### B. Gestión y ruteo de la Rutina de Interrupción (ISR)
La HAL utiliza un sistema unificado para centralizar el procesamiento de las interrupciones del microcontrolador y delegar el evento a la aplicación mediante funciones de retorno (callbacks):
1. **Mapeo de la ISR en la Tabla de Vectores:** Cuando TIM2 llega a su límite de cuenta (`4200`), el hardware de la CPU salta a la dirección asociada a `TIM2_IRQHandler` definida en `stm32f4xx_it.c`.
2. **Delegación al Manejador Común de la HAL:** Dentro de la rutina, la aplicación hace un llamado directo a la función de la HAL:
   ```c
   void TIM2_IRQHandler(void)
   {
     HAL_TIM_IRQHandler(&htim2);
   }
   ```
3. **Procesamiento de Registros por la HAL (`HAL_TIM_IRQHandler`):** Esta función común de la biblioteca HAL inspecciona los registros de estado del periférico para:
   - Verificar qué tipo de evento disparó la interrupción (por ejemplo, desbordamiento, captura/comparación, etc.).
   - Limpiar automáticamente las banderas de interrupción correspondientes en el hardware (como el bit `UIF` del registro `TIM2->SR`) para que la interrupción no se dispare de forma recursiva infinita.
   - Si se confirma un desbordamiento del contador ("Period Elapsed"), invoca a la función de retorno definida en la aplicación: `HAL_TIM_PeriodElapsedCallback(htim)`.

### C. Ejecución de la Acción del Usuario (Callback)
La HAL proporciona la función `HAL_TIM_PeriodElapsedCallback()` como un punto de inserción débil (`__weak`) para que el usuario escriba su lógica. En este proyecto, dicha función se encuentra implementada en `main.c` (línea 375):
```c
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim)
{
  // Comprobación de que la interrupción proviene de la instancia TIM2
  if (htim->Instance == TIM2)
  {
    // Lógica requerida por la aplicación para alimentar a FreeRTOS
    ulHighFrequencyTimerTicks++;
  }
}
```
* **Para qué sirve esta interacción:** Permite que TIM2 permanezca completamente administrado por los mecanismos estándar de ST para el control de temporizadores (ahorrando código complejo de manejo manual de registros y banderas de hardware), al mismo tiempo que permite desacoplar la base de tiempo de alta frecuencia del núcleo del kernel de FreeRTOS, garantizando un código portable, legible y estructurado bajo los estándares del fabricante.

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


