# Actividad 3 - paso 2

## ¿Cómo usar el parámetro de Tarea?

La unica forma de poder utilzarlo es castear este puntero void al tipo del parametro pasado. Por ejemplo:

```c
void task_btn(void *parameters) {
    .......
	global_dta_t* para = (global_dta_t*)parameters; // Castea los parametros al tipo esperado
    .......
}
```

Desde el lado de la inicialización, se debe pasar tal que:

```c
ret = xTaskCreate(task_led,
                  "Task LED1",
                  (2 * configMINIMAL_STACK_SIZE),
                  (void*)&global1_dta, // <== Pasamos la estructura como puntero a void
                  (tskIDLE_PRIORITY + 2ul),
                  &h_task_led1);
```

## ¿Cómo cambiar la prioridad de una Tarea ya creada?

Se usa `vTaskPrioritySet()` que permite modificar la prioridad de las tareas tal que:

```c
// Prototipo
void vTaskPrioritySet(TaskHandle_t xTask, UBaseType_t uxNewPriority);
```

Que podemos llamar dentro de una misma tarea para que se cambia a si misma la prioridad configurando `xTask` como `NULL`.

# Actividad 3 - paso 3 y 4

Como ambos ejercicios son similares, decidimos mostrarlos juntos. En `app.c` se crearon las dos instancias de los botones y de los leds tal que:

```c
/* Task BTN thread at priority 1 */
ret = xTaskCreate(task_btn,							/* Pointer to the function thats implement the task. */
                  "Task BTN1",						/* Text name for the task. This is to facilitate debugging only. */
                  (2 * configMINIMAL_STACK_SIZE),	/* Stack depth in words. */
                  (void*)&global1_dta,								/* We are not using the task parameter. */
                  (tskIDLE_PRIORITY + 1ul),			/* This task will run at priority 1. */
                  &h_task_btn1);						/* We are using a variable as task handle. */

/* Check the thread was created successfully. */
configASSERT(pdPASS == ret);

ret = xTaskCreate(task_btn,							/* Pointer to the function thats implement the task. */
                  "Task BTN2",						/* Text name for the task. This is to facilitate debugging only. */
                  (2 * configMINIMAL_STACK_SIZE),	/* Stack depth in words. */
                  (void*)&global2_dta,								/* We are not using the task parameter. */
                  (tskIDLE_PRIORITY + 1ul),			/* This task will run at priority 1. */
                  &h_task_btn2);						/* We are using a variable as task handle. */

/* Check the thread was created successfully. */
configASSERT(pdPASS == ret);

/* Task LED thread at priority 1 */
ret = xTaskCreate(task_led,							/* Pointer to the function thats implement the task. */
                  "Task LED1",						/* Text name for the task. This is to facilitate debugging only. */
                  (2 * configMINIMAL_STACK_SIZE),	/* Stack depth in words. */
                  (void*)&global1_dta,								/* We are not using the task parameter. */
                  (tskIDLE_PRIORITY + 2ul),			/* This task will run at priority 1. */
                  &h_task_led1);						/* We are using a variable as task handle. */

/* Check the thread was created successfully. */
configASSERT(pdPASS == ret);

ret = xTaskCreate(task_led,							/* Pointer to the function thats implement the task. */
                  "Task LED2",						/* Text name for the task. This is to facilitate debugging only. */
                  (2 * configMINIMAL_STACK_SIZE),	/* Stack depth in words. */
                  (void*)&global2_dta,								/* We are not using the task parameter. */
                  (tskIDLE_PRIORITY + 2ul),			/* This task will run at priority 1. */
                  &h_task_led2);						/* We are using a variable as task handle. */

/* Check the thread was created successfully. */
configASSERT(pdPASS == ret);
```
Cabe aclarar que ahora cada boton controla un led distinto, de modo que cada tarea de boton esta asociada a una tarea de led.

Donde a cada tarea se le pasa como parametro la siguiente estructura:

```c
typedef struct {
	task_btn_dta_t task_btn_dta; // button handler
	task_led_dta_t task_led_dta; // led handler
} global_dta_t;
```

Ademas, se tuvo que modificar ambas tareas para castear este parametro y utilizarolo en las FSMs que ahora reciben el parametro del boton/led:

```c
void task_btn(void *parameters)
{
    // .........
	global_dta_t* para = (global_dta_t*)parameters;
	task_btn_dta_t* aux = &(para->task_btn_dta);
    // .........
	for (;;)
	{
        // Tuvimos que cambiar la fsm para que use estos parametros especificos a cada tarea
    	task_btn_statechart(aux,&(para->task_led_dta));
	}
}
```

Esto mismo se hizo para la tarea de led de modo que ambas tareas reciban los datos del boton y led correspondiente. Ademas, se tuvo que cambiar en `task_led_interface.c` la funcion `put_event_task_led` para que reciba el handler del led en vez de usar una variable global.

```c
void put_event_task_led(task_led_ev_t event)
{
	task_led_dta.event = event;
	task_led_dta.flag = true;
}
```

Finalmente, como se puede ver en las prioridades de las tareas de led, ambas comienzan con `taskIDLE_PRIORITY + 2ul` que es mayor a las tareas de boton y despues dentro de la tarea de led se cambia tal que:

```c
vTaskPrioritySet(NULL, tskIDLE_PRIORITY + 1ul);
```
Esto se llama una vez en el comienzo de la tareas de led para bajarles la prioridad.

# Ejemplo andando con los dos botones

A modo de muestra, armamos un video con los dos botones andando:

