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
# TODO : CONTESTAR MAS TARDE
# ¿Cómo enviar datos a una Cola?

```c
BaseType_t xQueueSend(QueueHandle_t xQueue, const void * pvItemToQueue, TickType_t xTicksToWait);
```

# ¿Cómo recibir datos de una Cola?

```c
BaseType_t xQueueReceive(QueueHandle_t xQueue, void *pvBuffer, TickType_t xTicksToWait);
```

# ¿Qué significa bloquearse en una Cola?
# TODO : CONTESTAR MAS TARDE
# ¿Cómo bloquearse en varias Colas?

```c
QueueSetHandle_t xQueueCreateSet(const UBaseType_t uxEventQueueLength);
```
# ¿Cómo sobrescribir datos en una Cola?
```c
BaseType_t xQueueOverwrite(QueueHandle_t xQueue, const void * pvItemToQueue);
```
# ¿Cómo vaciar una Cola?
```c
BaseType_t xQueueReset(QueueHandle_t xQueue);
```
# ¿Cuál es el efecto de las prioridades de las Tareas al escribir y leer en una Cola?
# TODO : CONTESTAR MAS TARDE
