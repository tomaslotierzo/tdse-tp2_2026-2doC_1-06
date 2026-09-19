A continuación, se presenta el análisis del código fuente proporcionado, detallando la función de cada archivo, el comportamiento de las variables clave y el funcionamiento de la máquina de estados y la cola de eventos.
Análisis General de los Archivos
task_sensor_attribute.h: Define los tipos de datos, estructuras (como task_sensor_cfg_t y task_sensor_dta_t) y enumeraciones (task_sensor_ev_t, task_sensor_st_t) necesarios para modelar las características y el estado interno del sensor (el botón).
task_system_attribute.h: Define las enumeraciones de eventos (task_system_ev_t) y estados (task_system_st_t), así como las estructuras de datos propias de la tarea del sistema (la tarea que consume los eventos del botón).
task_sensor.c: Implementa la lógica de la tarea del sensor. Contiene la configuración de los botones, la inicialización (task_sensor_init), el bucle de actualización (task_sensor_update) y la máquina de estados (task_sensor_statechart) que evalúa el hardware y genera eventos de software.
task_system_interface.c: Implementa una cola circular (FIFO) que actúa como canal de comunicación (interfaz) entre la tarea del sensor y la tarea del sistema. Permite almacenar eventos generados por el sensor para que el sistema los procese asíncronamente.
Evolución de las variables de task_sensor
Las siguientes variables evolucionan desde la llamada a task_sensor_init() y durante el bucle task_sensor_update():
index:
En task_sensor_init(), comienza en $0$ y finaliza el bucle sin llegar a iterar con el valor $1$, ya que la constante SENSOR_DTA_QTY es igual a $1$ (sólo hay un botón configurado).
En task_sensor_update(), ocurre exactamente lo mismo: evoluciona tomando únicamente el valor $0$ para procesar el único elemento del arreglo.
task_sensor_dta_list[index].tick:
Unidad de medida: Ticks de la aplicación (implícitamente milisegundos, dado que la tarea se actualiza cada 1 ms).
Evolución: No se inicializa explícitamente en task_sensor_init() y, en esta versión del código, no se incrementa ni se utiliza activamente para retardos en los estados regulares. Solo toma el valor DEL_BTN_MIN ($0$) si la máquina de estados cayera accidentalmente en el caso default.
task_sensor_dta_list[index].state:
En task_sensor_init(), se inicializa con el valor ST_BTN_IDLE (estado de reposo).
En sucesivas llamadas a task_sensor_update(), su valor cambia a ST_BTN_ACTIVE cuando el botón es presionado, y vuelve a ST_BTN_IDLE cuando el botón es liberado.
task_sensor_dta_list[index].event:
En task_sensor_init(), se inicializa forzadamente en EV_BTN_UP (botón sin pulsar).
En cada llamada a task_sensor_update(), se sobreescribe constantemente evaluando el nivel físico del pin (GPIO). Toma el valor EV_BTN_DOWN si el botón físico está apretado, o EV_BTN_UP si no lo está.
Comportamiento de la función task_sensor_statechart(uint32_t index)
Esta función implementa el comportamiento (máquina de estados finitos) de un sensor específico apuntado por index:
Lectura del Hardware (Polling): Utiliza HAL_GPIO_ReadPin para leer el estado eléctrico del pin configurado. Si el estado del pin coincide con el nivel lógico esperado para considerarse presionado (pressed), actualiza el evento local de la tarea a EV_BTN_DOWN; de lo contrario, lo pone en EV_BTN_UP. (Nota: esta implementación realiza una lectura directa y cruda sin lógica algorítmica de antirrebote/debounce).
Evaluación de Estados (Switch-Case):
Si el estado es ST_BTN_IDLE: Si detecta el evento EV_BTN_DOWN, despacha la señal de pulsación (signal_down o EV_SYS_ACTIVE) hacia la cola del sistema llamando a put_event_task_system(), y cambia su estado a ST_BTN_ACTIVE para no enviar el mismo evento repetidamente.
Si el estado es ST_BTN_ACTIVE: Espera a que el evento vuelva a ser EV_BTN_UP. Cuando esto sucede, despacha la señal de liberación (signal_up o EV_SYS_IDLE) a la cola del sistema y regresa al estado ST_BTN_IDLE.
El caso default actúa como mecanismo de recuperación, forzando la máquina a un estado seguro (ST_BTN_IDLE).
Evolución de las variables de la cola (event_task_system_queue)
Estas variables administran la cola circular de eventos implementada en task_system_interface.c. Aunque su inicialización formal ocurre al llamar a init_event_task_system() (probablemente en la inicialización de la tarea del sistema), su evolución ante las acciones de la tarea del sensor es la siguiente:
event_task_system_queue.count:
Comienza en $0$ (cola vacía).
Se incrementa en $+1$ cada vez que la tarea del sensor detecta un flanco (al presionar o soltar el botón) y llama a put_event_task_system().
event_task_system_queue.head (Cabeza/Escritura):
Comienza apuntando al índice $0$.
Cuando el sensor inserta un nuevo evento llamando a put_event_task_system(), head se incrementa para apuntar al próximo bloque libre.
Si alcanza el límite del arreglo (QUEUE_LENGTH, que es $16$), vuelve automáticamente a $0$ para reusar el principio del buffer (comportamiento circular).
event_task_system_queue.tail (Cola/Lectura):
Comienza en el índice $0$.
No evoluciona ni se modifica por las acciones del sensor; solo avanza (incrementándose y volviendo a $0$ al igual que el head) cuando la tarea del sistema (consumidora) llama a la función get_event_task_system() para retirar y procesar los mensajes.
event_task_system_queue.queue[i] (Buffer de datos):
Inicialmente, todas las 16 posiciones (de $0$ a $15$) se llenan con la macro EMPTY ($255$).
Cuando el sensor llama a put_event_task_system(), la posición actual indicada por head (queue[head]) abandona el valor EMPTY y pasa a almacenar el valor numérico del evento enviado (EV_SYS_ACTIVE o EV_SYS_IDLE).
Permanecerá con ese valor hasta que la tarea consumidora extraiga el dato, momento en el cual la posición leída (indicada por tail) será sobreescrita nuevamente con el valor EMPTY para quedar libre.
