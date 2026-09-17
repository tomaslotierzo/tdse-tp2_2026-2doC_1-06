A continuación, se presenta el análisis del código encargado de coordinar la lógica principal del sistema, sirviendo de puente entre los estímulos (sensores) y las respuestas (actuadores).
Evolución de las variables del Sistema
En el archivo task_system.c, la información de la tarea central se almacena en el arreglo task_system_dta_list.
index: Durante task_system_init(), toma el valor 0 (correspondiente a NORMAL) para recorrer e inicializar la única instancia del sistema.
tick: Su unidad de medida implícita son los milisegundos (1 ms por tick). No se incrementa activamente, pero se asigna a 0 (DEL_SYS_MIN) si el flujo cae en la condición default.
state: Inicializa en ST_SYS_IDLE. Alternará entre este y ST_SYS_ACTIVE según los mensajes que procese.
event y flag: Inician en EV_SYS_IDLE y false respectivamente. Cuando el sistema lee un dato de la cola, event adopta el mensaje recibido y flag pasa a true. Al consumirse en la máquina de estados, flag regresa a false.
Comportamiento de task_system_normal_statechart(void)
Aunque tu consulta menciona task_sensor_statechart(uint32_t index), en este módulo la función equivalente es task_system_normal_statechart(void) (ya que el índice es fijo en modo NORMAL).
Verifica si hay eventos en la cola mediante any_event_task_system(). Si existen, los extrae con get_event_task_system() y levanta su bandera (flag).
Estando en ST_SYS_IDLE: Si lee el evento EV_SYS_ACTIVE, apaga la bandera, envía el evento EV_LED_ACTIVE al actuador mediante put_event_task_actuator(), y transiciona al estado ST_SYS_ACTIVE.
Estando en ST_SYS_ACTIVE: Si recibe EV_SYS_IDLE, despacha EV_LED_IDLE al actuador, apaga la bandera y retorna al reposo.
Evolución de la Cola de Eventos
La estructura event_task_system_queue permite la comunicación asíncrona.
i y queue[i]: i se usa en init_event_task_system() para recorrer de 0 a 15 y llenar la matriz queue con el valor EMPTY (255). Las celdas almacenarán los eventos hasta que la lectura los devuelva a EMPTY.
head y tail: Ambos inician en 0. head avanza cuando otros módulos insertan datos, mientras que tail lo persigue al ejecutarse las lecturas del sistema. Ambos regresan a 0 al llegar a 16.
count: Sube con ingresos y baja con egresos, indicando la cantidad de elementos pendientes.
Evolución de las variables del Actuador
Al ejecutar put_event_task_actuator() en task_system.c, se manipula la estructura task_actuator_dta_list.
identifier: Toma el valor fijo ID_LED_A (indicando qué LED controlar).
event y flag: event adquiere la orden indicada (EV_LED_ACTIVE o EV_LED_IDLE), y flag pasa a true para avisar al módulo del actuador que tiene un trabajo pendiente por resolver en su próxima actualización.
¿Te gustaría profundizar en cómo el código específico de la tarea del actuador procesa estas señales y manipula el hardware?
