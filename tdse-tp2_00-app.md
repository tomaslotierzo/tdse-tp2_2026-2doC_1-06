El sistema proporcionado implementa una arquitectura de tiempo real Bare Metal orientada a eventos por disparo de tiempo (ETS - Event-Triggered Systems) sobre un microcontrolador ARM Cortex-M.
Funcionamiento General del Código Fuente
app.c: Es el núcleo de la aplicación; administra un arreglo de tareas (task_sensor, task_system, task_actuator) ejecutadas periódicamente cada 1 ms en app_update(), midiendo los tiempos de ejecución de cada una mediante el periférico DWT.
app_it.c: Maneja la rutina de interrupción de tiempo HAL_SYSTICK_Callback(), la cual incrementa el contador global de ticks de la aplicación g_app_tick_cnt.
systick.c: Ofrece la función de retardo bloqueante systick_delay_us() basada en la lectura directa del registro de cuenta del SysTick.
logger.c y logger.h: Implementan el sistema de traza e impresión de mensajes de depuración a través de semihosting mediante macros bloqueantes (LOGGER_LOG, LOGGER_INFO) que deshabilitan interrupciones.
dwt.h: Proporciona funciones en línea (inline) para inicializar, reiniciar y leer el contador de ciclos del DWT (Data Watchpoint and Trace) del procesador para convertir ciclos en microsegundos.
Variables y Unidades de Medida

Variable
Unidad de Medida
Descripción
g_app_tick_cnt
Ticks / Ocurrencias (1 ms por tick)
Contador de interrupciones de tiempo pendientes.
g_app_runtime_us
Microsegundos ($\mu\text{s}$)
Tiempo total de ejecución de todas las tareas en el tick actual.
index
Adimensional / Índice ($0, 1, 2$)
Índice de la matriz de tareas.
task_dta_list[index].NOE
Adimensional / Numeral
Número de veces que se ha ejecutado la tarea (Number of Executions).
task_dta_list[index].LET
Microsegundos ($\mu\text{s}$)
Último tiempo de ejecución registrado (Last Execution Time).
task_dta_list[index].BCET
Microsegundos ($\mu\text{s}$)
Mejor tiempo de ejecución registrado (Best-Case Execution Time).
task_dta_list[index].WCET
Microsegundos ($\mu\text{s}$)
Peor tiempo de ejecución registrado (Worst-Case Execution Time).

Evolución de las Variables durante la Ejecución
1. Fase de Inicialización (app_init() en app.c)
g_app_tick_cnt: Se reinicia a $0$ dentro de app_it_init() y comienza a incrementarse asíncronamente mediante HAL_SYSTICK_Callback() una vez habilitadas las interrupciones.
index: Toma secuencialmente los valores $0$, $1$ y $2$ dentro del bucle de inicialización de tareas.
task_dta_list[index].NOE: Se fija en $0$ (TASK_X_NOE_INI) para cada una de las 3 tareas.
task_dta_list[index].LET: Se establece en $0\,\mu\text{s}$ (TASK_X_LET_INI) para cada tarea.
task_dta_list[index].BCET: Se fija en un límite superior inicial de $1000\,\mu\text{s}$ (TASK_X_BCET_INI) para permitir la posterior captura del menor tiempo real registrado.
task_dta_list[index].WCET: Se fija en $0\,\mu\text{s}$ (TASK_X_WCET_INI) para cada tarea.
g_app_runtime_us: Permanece sin modificar hasta el ingreso al bucle principal.
2. Fase de Bucle Principal (app_update() en app.c)
g_app_tick_cnt: Si es mayor que $0$, se decrementa en $1$ (en sección crítica) y activa la bandera de actualización.
g_app_runtime_us: Al iniciar la atención de un tick se reinicia a $0\,\mu\text{s}$. Al final del recorrido de las tareas, contiene la suma acumulada del tiempo LET de cada una de ellas ($\sum \text{LET}$) en ese tick particular.
index: Evoluciona dentro del ciclo for desde $0$ hasta $2$, haciendo referencia secuencial a task_sensor, task_system y task_actuator.
task_dta_list[index].NOE: Se incrementa en $+1$ en cada pasada del bucle para la tarea correspondiente ($1, 2, 3, \dots$).
task_dta_list[index].LET: Al finalizar task_update, almacena la duración en microsegundos calculada desde cycle_counter_get_time_us() para esa ejecución específica.
task_dta_list[index].BCET: Si el LET actual es menor que el BCET almacenado, BCET se actualiza con el valor de LET.
task_dta_list[index].WCET: Si el LET actual es mayor que el WCET almacenado, WCET se actualiza con el valor de LET.
Impacto del uso de LOGGER_INFO() en las Variables
La macro LOGGER_INFO() deshabilita las interrupciones del sistema (CPSID i), formatea la cadena con snprintf() y la envía a través de semihosting (printf() y fflush()), lo que constituye una operación de entrada/salida altamente bloqueante y lenta.
Impacto en task_dta_list[index].WCET: Si se invoca LOGGER_INFO() dentro de la función de actualización de una tarea, el contador de ciclos del periférico DWT (CYCCNT) seguirá contando activamente. Como consecuencia, el LET de esa ejecución aumentará drásticamente en cientos o miles de microsegundos, lo que provocará que WCET absorba dicho pico y registre un peor tiempo de ejecución ficticio e inflado por la sobrecarga (overhead) del logger.
Impacto en g_app_runtime_us: Dado que g_app_runtime_us es la suma acumulativa de los tiempos LET de todas las tareas (g_app_runtime_us += LET), la llamada a LOGGER_INFO() incrementará de forma directa el tiempo total de ejecución registrado para la aplicación durante ese ciclo de trabajo de 1 ms.
