# Actividad 2 - Paso 2

## ¿Cómo asigna tiempo de procesamiento a cada tarea?

Lo hace con el SysTick timer interrumpiendo las tareas y verificando las prioridades para delegar el control de la CPU a la tarea correspondiente.

## ¿Cómo FreeRTOS elige qué Tarea debe ejecutarse en un momento dado?

Lo hace en base al nivel de prioridad de cada tarea.

## ¿Cómo la prioridad relativa de cada Tarea afecta el comportamiento del sistema?

Afecta haciendo que aquella de mayor prioridad se ejecute idefinidamente si no se pone en estado suspend o blocked.
En cambio en caso de tener igual prioridad iran cambiando entre las dos equitativamente de forma Round Robin.

## ¿Cuáles son los estados en los que puede encontrarse una Tarea?

Running, ready, suspend y blocked.

## ¿Cómo implementar Tareas?

Cada tarea se debe implementar con una funcion en C.

## ¿Cómo crear una o más instancias de una Tarea?

Para crear una tarea se debe usar la funcion:

```c
xTaskCreate(<Pointer to the task function>,
            <Task string name id>,
            <Reserved memory for the stack>,
            <Pointer to the parameters>,
            <Priority level>,
            <Pointer to a handler function>);
```

En caso de crear mas de una tarea, se debe llamar repetidas veces a esta funcion con diferentes parametros.

## ¿Cómo eliminar una Tarea? 

Para eliminar una tarea se hace lo siguiente:


```c
vTaskDelete(<task handler>);
vTaskDelete(NULL); // Elimina la tarea que llama a la funcion
```

# Actividad 2 - Paso 3

Se aumento la prioridad a la tarea del boton por lo que se observo que la tarea del led nunca se ejecuto:

```
[info] Task BTN is running - Tick [mS] =   0
[info]  Task BTN - BTN PRESSED
[info]  Task BTN - BTN HOVER
[info]  Task BTN - BTN PRESSED
[info]  Task BTN - BTN HOVER
```

Esto es debido a que la tarea del boton nunca suelta el recurso y por lo tanto se ejecuta indefinidamente.

# Activad 2 - paso 4

Se agregaron dos tareas de botones con la misma función y prioridad, pero distintos handlers. Lo esperado es que al compartir la misma prioridad, el scheduler hace time-slicing y se ejecutan durante un systick cada una.

```C
    TaskHandle_t h_task_btn1;
    TaskHandle_t h_task_btn2;
    TaskHandle_t h_task_btn3;
    TaskHandle_t h_task_led;

    // ..... Some more code

    /* Task BTN thread at priority 1 */
    ret = xTaskCreate(task_btn,							/* Pointer to the function thats implement the task. */
					  "Task BTN1",						/* Text name for the task. This is to facilitate debugging only. */
					  (2 * configMINIMAL_STACK_SIZE),	/* Stack depth in words. */
					  NULL,								/* We are not using the task parameter. */
					  (tskIDLE_PRIORITY + 1ul),			/* This task will run at priority 1. */
					  &h_task_btn1);						/* We are using a variable as task handle. */

    /* Task BTN thread at priority 1 */
	ret = xTaskCreate(task_btn,							/* Pointer to the function thats implement the task. */
					  "Task BTN2",						/* Text name for the task. This is to facilitate debugging only. */
					  (2 * configMINIMAL_STACK_SIZE),	/* Stack depth in words. */
					  NULL,								/* We are not using the task parameter. */
					  (tskIDLE_PRIORITY + 1ul),			/* This task will run at priority 1. */
					  &h_task_btn2);

    // ..... Some more code

    /* Task LED thread at priority 1 */
    ret = xTaskCreate(task_led,							/* Pointer to the function thats implement the task. */
					  "Task LED",						/* Text name for the task. This is to facilitate debugging only. */
					  (2 * configMINIMAL_STACK_SIZE),	/* Stack depth in words. */
					  NULL,								/* We are not using the task parameter. */
					  (tskIDLE_PRIORITY + 1ul),			/* This task will run at priority 1. */
					  &h_task_led);						/* We are using a variable as task handle. */
```

Por otro lado, se agrego a la tarea de led la siguiente linea antes del loop. Esto permite que en el primer llamado se borre la tarea:

```C
	vTaskDelete(h_task_btn3);
```

Finalmente, cuando se hizo la prueba se observo que funcionaba correctamente la mayor parte del tiempo dando el siguiente log:
```
    [info]  Task BTN1 - BTN PRESSED
    [info]  Task LED - LED BLINK
    [info]  Task BTN1 - BTN HOVER
    [info]  Task LED - LED OFF
    [info]  Task BTN1 - BTN PRESSED
    [info]  Task LED - LED BLINK
    [info]  Task BTN1 - BTN HOVER
    [info]  Task LED - LED OFF
```

Sin embargo, a veces ocurria lo siguiente:

```
[info]  Task BTN1 - BTN PRESSED
[info]  Task BTN2 - BTN PRESSED  <- DUPLICADO POR SEGUNDA TAREA
[info]  Task LED - LED BLINK
[info]  Task BTN1 - BTN HOVER
[info]  Task BTN2 - BTN HOVER    <- DUPLICADO POR SEGUNDA TAREA
[info]  Task LED - LED OFF
```
Esto es producto de que ambas tareas utilizan una maquina de estados que usa una variable global privada `task_btn_dta` por lo que ambas la modifican y se "pisan" en la ejecución.

Por otro lado, usando algunos breakpoints pudimos comprobar que la tercer tarea del boton se llego a crear:
![Prev to delete](TP1/sotri-tp1_02-application/images/prev.png)

Posterior al delete:
![Post delete](TP1/sotri-tp1_02-application/images/post.png)

