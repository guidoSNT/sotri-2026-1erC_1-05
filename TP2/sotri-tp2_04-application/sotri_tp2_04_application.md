# ¿Qué funciones de la API de FreeRTOS se pueden usar dentro de una rutina de servicio de interrupción?

En este caso utilizamos:

```c
 SemaphoreGiveFromISR(h_btn_led_bin_sem, &xHigherPriorityTaskWoken);
```

En general, para FreeRTOS todas las que terminen con ISR se podran usar en interrupciones.

# ¿Métodos para delegar el procesamiento de interrupciones a una Tarea?

Cada metodo de comunicación entre tareas (semaforos, queues, notifies, etc) tienen sus funciones correspondientes para comunicar desde interrupciones (geneneralmente terminan con ISR).

# ¿Cómo usar una cola para transferir datos dentro y fuera de una rutina de servicio de interrupción?

Para interactuar con una `queue` desde una interrupción se puede usar las siguientes funciones:

```c
xQueueSendFromISR() // Suma un elemento al queue
xQueueReceiveFromISR() // Obtiene el primer elemento del queue
```

# ¿Cuál es el modelo de anidamiento de interrupciones disponible en algunas portaciones de FreeRTOS?

Existe un umbral determinado por `configMAX_SYSCALL_INTERRUPT_PRIORITY` en donde las interrupciones con mayor prioridad que este, no son deshabilitadas por el FreeRTOS.
En cambio las que se encuentras debajo de este umbral, si estan bajo la gestion de FreeRTOS y son deshabilitadas temporalmente cuando el kernel ejecuta alguna sección critica.
