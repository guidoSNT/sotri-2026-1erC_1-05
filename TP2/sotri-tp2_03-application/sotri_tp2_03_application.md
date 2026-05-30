# ¿Cómo crear y usar semáforos binarios y semáforos contadores?

para crear semaforos binarios o contadores se deben usar las siguientes funciones:

```c
 SemaphoreHandle_t xSemaphoreCreateBinary(void) // Para crear semaforos binarios
 SemaphoreHandle_t xSemaphoreCreateCounting(UBaseType_t uxMaxCount, UBaseType_t uxInitialCount); // Para crear semaforos contadores

```
## Semáforo Binario (Sincronización):

Se usa para que una tarea espere a que ocurra un evento (como una interrupción).

+ (Crear): xHandler = xSemaphoreCreateBinary();

+ (Esperar/Tomar): La tarea llama a xSemaphoreTake(). Se quedará dormida (bloqueada) ahí hasta que el evento ocurra.

+ (Avisar/Dar): Cuando ocurre el evento (ej. llega un dato por Serial), se llama a xSemaphoreGive(). Esto "despierta" a la tarea.

## Semáforo Contador (Recursos o Conteo)

Se usa cuando tienes varios recursos iguales o quieres contar eventos acumulados.

Paso 1 (Crear): xHandler = xSemaphoreCreateCounting(max_cuenta, inicial);

Paso 2 (Usar/Tomar): Cada vez que una tarea usa un recurso, llama a xSemaphoreTake(). El contador resta 1. Si llega a 0, la siguiente tarea espera.

Paso 3 (Liberar/Dar): Al terminar de usar el recurso, la tarea llama a xSemaphoreGive(). El contador suma 1.

# ¿Cuáles son las diferencias entre semáforos binarios y semáforos contadores?

La diferencia es la siguiente:

- `Semaforo binario`:
    - Puede tomar valor 0 o 1. 
    - Un give lo cambia a uno.
    - Un take lo cambia a cero.
    - Sucesivos gives no tienen efecto.
    - A partir del primer take todos los demás bloquean la tarea.

- `Semaforo contador`:
    - Puede tomar valor de 0 o N. 
    - Un give aumenta el contador (con limite hasta N).
    - Un take disminuye el contador.
    - Una vez el contador llega a 0 todos los takes bloquean las tareas.
