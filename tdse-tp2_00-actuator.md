A continuación, te presento el análisis detallado del funcionamiento del código encargado de controlar los actuadores del sistema (en este caso, un LED).
Función General de los Archivos
task_actuator_attribute.h: Define la estructura de datos interna del actuador. Incluye los eventos que puede recibir (task_actuator_ev_t), sus posibles estados (task_actuator_st_t), y las estructuras para almacenar tanto su configuración estática referida al hardware (task_actuator_cfg_t) como su contexto dinámico (task_actuator_dta_t).
task_actuator_interface.c: Expone un mecanismo de comunicación directo (sin colas) para que otras tareas (como la del sistema) puedan enviarle órdenes al actuador.
task_actuator.c: Contiene la implementación lógica de la tarea. Aquí se define la configuración inicial, la rutina de inicialización, el ciclo de actualización periódico y la máquina de estados que reacciona a los comandos modificando el hardware mediante la capa HAL.
Evolución de las variables de la Tarea del Actuador
Al ejecutar task_actuator_init() y posteriormente las sucesivas llamadas a task_actuator_update(), las variables internas del arreglo task_actuator_dta_list evolucionan de la siguiente manera:
index:
Tanto en la inicialización como en la actualización, esta variable se utiliza en un bucle for para iterar sobre todos los actuadores configurados.
Dado que ACTUATOR_DTA_QTY es $1$ (solo está configurado ID_LED_A), index toma el valor $0$, procesa ese elemento, y el bucle finaliza.
task_actuator_dta_list[index].tick:
Unidad de medida: Milisegundos (ms), deducido por la frecuencia de actualización de la tarea indicada en los comentarios (period = 1mS).
En esta implementación específica, la variable no se incrementa activamente de forma periódica; solo se reinicia al valor DEL_LED_MIN ($0$) si la máquina de estados cae en un fallo (caso default).
task_actuator_dta_list[index].state:
Durante task_actuator_init(), se establece de forma inicial en ST_LED_IDLE.
Durante la ejecución de task_actuator_update(), cambiará a ST_LED_ACTIVE cuando reciba la orden de encendido, y regresará a ST_LED_IDLE al recibir la orden de apagado.
task_actuator_dta_list[index].event:
Se inicializa con el valor EV_LED_IDLE.
Su valor se modifica asíncronamente a EV_LED_ACTIVE o EV_LED_IDLE cuando un agente externo invoca la interfaz del actuador.
task_actuator_dta_list[index].flag:
Se inicializa en false.
Pasa a true cuando se registra un nuevo evento a través de la interfaz.
Regresa a false inmediatamente después de que la máquina de estados procesa y consume dicho evento.
Comportamiento de task_actuator_statechart(uint32_t index)
Esta función actúa como el "cerebro" del actuador evaluando su estado actual y determinando si debe cambiar el estado físico del pin:
Actualiza los punteros locales para acceder a la configuración de hardware y los datos dinámicos del actuador indicado por index.
Evalúa en qué estado se encuentra mediante un switch:
Si está en ST_LED_IDLE: Espera a que haya un evento pendiente (flag == true) y que la orden sea encender (EV_LED_ACTIVE). Si se cumple, baja la bandera (flag = false), escribe el nivel lógico definido como led_on en el pin asociado usando HAL_GPIO_WritePin, y transiciona al estado ST_LED_ACTIVE.
Si está en ST_LED_ACTIVE: Espera a que haya un evento pendiente (flag == true) y que la orden sea apagar (EV_LED_IDLE). Al cumplirse, baja la bandera, apaga el LED escribiendo led_off en el pin físico, y regresa al estado ST_LED_IDLE.
En caso de caer en un estado no reconocido (default), ejecuta una rutina de seguridad: resetea el tick a $0$, fuerza el estado a ST_LED_IDLE, borra los eventos y apaga la bandera.
Evolución de las variables de Interfaz (task_actuator_interface.c)
A diferencia de la tarea del sistema que usa una cola circular, el actuador utiliza un mecanismo de buzón directo (sobrescritura) a través de la función put_event_task_actuator(task_actuator_ev_t event, task_actuator_id_t identifier).
identifier: Es el argumento que recibe la función para indicar a qué actuador va dirigido el mensaje. En esta aplicación toma el valor ID_LED_A ($0$). No evoluciona, depende de quién llame a la función.
task_actuator_dta_list[identifier].event: Cuando se ejecuta la función de interfaz, esta variable es sobrescrita de forma inmediata con el valor del parámetro event ingresado (EV_LED_ACTIVE o EV_LED_IDLE), dejando el mensaje listo para que la máquina de estados lo evalúe en su próxima iteración.
task_actuator_dta_list[identifier].flag: Cada vez que se ejecuta la interfaz y se carga un nuevo evento, se fuerza a true. Esto le avisa a task_actuator_statechart que tiene un mensaje fresco esperando ser procesado y que debe actuar en consecuencia.
¿Hay algún punto específico sobre la interacción entre la interfaz de eventos y el hardware sobre el que quisieras profundizar?
