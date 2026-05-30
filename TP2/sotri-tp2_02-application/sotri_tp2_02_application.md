# ¿Cómo eliminar una Cola?

Para eliminar una cola se puede llamar a esta función para que la elimine.
```c
void vQueueDelete( QueueHandle_t xQueue);
```
# ¿Cómo crear una Cola?

```c
QueueHandle_t xQueueCreate( UBaseType_t uxQueueLength, UBaseType_t uxItemSize );
```
# ¿Cómo gestiona una Cola los datos que contiene?

Una cola en FreeRTOS es un buffer circular generalmente FIFO o LIFO, con el que podemos quitar o incorporar elementos desde cualquier tarea por copia.

# ¿Cómo enviar datos a una Cola?

```c
BaseType_t xQueueSend(QueueHandle_t xQueue, const void * pvItemToQueue, TickType_t xTicksToWait);
```

# ¿Cómo recibir datos de una Cola?

```c
BaseType_t xQueueReceive(QueueHandle_t xQueue, void *pvBuffer, TickType_t xTicksToWait);
```

# ¿Qué significa bloquearse en una Cola?

Significa que la tarea se queda esperando a poder recivir un elemento (si esta vacia) o a poder incorporar uno (si esta llena).

# ¿Cómo bloquearse en varias Colas?

En caso que se creen multiples colas y queramos esperar a recibir de alguna de ellas, podemos usar la siguiente función: 
```c
QueueSetHandle_t xQueueCreateSet(const UBaseType_t uxEventQueueLength);
```
Esta función crea un `Set` para que al usar `xQueueSelectFromSet()` se bloquee hasta recibir en alguna de todas las colas.

# ¿Cómo sobrescribir datos en una Cola?
```c
BaseType_t xQueueOverwrite(QueueHandle_t xQueue, const void * pvItemToQueue);
```
# ¿Cómo vaciar una Cola?
```c
BaseType_t xQueueReset(QueueHandle_t xQueue);
```
# ¿Cuál es el efecto de las prioridades de las Tareas al escribir y leer en una Cola?

Si hay varias tareas bloqueadas esperando a una queue, aquella tarea de mayor prioridad será la primera en salir sin importar cuanto tiempo hayan esperado las otras tareas.
Sin embargo, si todas las tareas que esperan son de la misma prioridad, recibe la primera que llamo al send o receive.
