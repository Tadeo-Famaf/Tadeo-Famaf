# INFORME XV6-RISCV

## Primera Parte

### **¿Qué política de planificación utiliza `xv6-riscv` para elegir el próximo proceso a ejecutarse?**

Xv6-riscv usa *RR (Round Robin)* como política de planificación. La implementación la podemos encontrar en el directorio `so24lab3g17/kernel/proc.c`, en específico en la función `scheduler()`.

**Así funciona:**

![scheduler](https://github.com/user-attachments/assets/ad009c6f-4e06-493c-8e60-23180789b72c)



Podemos observar que la función scheduler contiene 2 loops, el primer loop (`for(;;)`) es infinito, esto para buscar continuamente procesos que puedan ejecutarse, luego dentro de este loop se habilitan las interrupciones con `intr_on()` para evitar un bloqueo si los procesos están en espera.

En el segundo loop (`for(p = proc; p < &proc[NPROC]; p++)`) se itera sobre la lista de procesos, después se adquiere el lock (o bloqueo) del proceso p (`acquire(&p->lock)`) y a continuación se verifica si
el proceso p se encuentra en estado `RUNNABLE`, lo que significa que está listo para ejecutarse.

Si el proceso está en RUNNABLE, se procede a cambiarle el estado a `RUNNING` para indicar que este nuevo proceso p está en ejecución, luego se guarda el proceso actual (p) en el contexto de la CPU y se llama a la función `swtch`
para realizar un cambio de contexto entre el proceso que se **estaba** ejecutando y el **nuevo** proceso p y así se ejecuta por fin el proceso p.

Posteriormente, se libera el lock (`release(&p->lock)`).



***

### **¿Cuáles son los estados en los que un proceso puede permanecer en xv6-riscv y qué los hace cambiar de estado?**


![procstate](https://github.com/user-attachments/assets/6d901657-ee44-4028-8b54-530af2438bcc)


Los estados posibles en los que puede permanecer xv6-riscv son:
 - **UNUSED**: esta es una entrada de la process table (tabla de procesos) esperando a ser asignado a un nuevo proceso, y por lo tanto no está siendo utilizada ni está activa hasta que suceda dicha asignación. Por esto podemos decir que *un proceso en este estado es inexistente ya que aún no es computable*. De cierta manera tambien podriamos decir que los procesos en un sub-estado **TERMINATED** se encuentran aquí también ya que estos son procesos que no ocupan un lugar particular en la process table ya que dicha información a sido liberada una vez terminada su labor.
 - **USED**: cuando a la anterior entrada se le asigna un nuevo proceso; este cambia su estado a *USED*, es decir que dicha entrada ya está en utilización e inicializada con su correspondiente información ; *por ende no es un estado en sí; sino que nos demuestra que está activo* (es decir su estado puede ser SLEEPING, RUNNABLE ó RUNNING).
 - **SLEEPING**: es un proceso el cual está esperando por algún recurso que este pidió; ya se una llamada a alguna **I/O** (entrada ó salida) o aspecto externo a su ejecución misma (por ejemplo que su CHILD termine de ejecutar); *por esto podemos decir que es un proceso que se encuentra temporalmente sin ejecutar.*
 - **RUNNABLE**: es un proceso que está esperando en la cola de ejecución su turno de CPU; podría estar ejecutándose pero debido a que *no tiene acceso a una CPU debido a la alta demanda de la misma debe; permanecer en este estado hasta que el scheduler lo asigne en un núcleo*.
 - **RUNNING**: *son los procesos que se encuentran en dicho momento ocupando una CPU y están siendo ejecutados* hasta que una interrupción suceda o necesite de un recurso externo a sí mismo.
 - **ZOMBIE**: *son procesos que han finalizado su ejecución y no tienen nada más para hacer*; pero están esperando la llamada de sus FATHERS para que se libere su información de la process table.
 
 Estos procesos están declarados en `so24lab3g17/kernel/proc.h` como se ve a continuación:
"foto del proc.h"


**¿Qué hace un proceso para cambiar de un estado al otro?**
 Bueno la respuesta es un poco compleja ya que los motivos son varios; pero principalmente podríamos decir que sucede un context switch en el cual ya sea por interrupción o término del CPU-TIME, por errores excepcionales, syscalls o por la misma planificación del scheduler. Por lo general podemos decir que los cambios ocurren entre los siguientes estados:
 - UNUSED a USED (RUNNABLE).
 - RUNNABLE a RUNNING.
 - RUNNING a SLEEPING.
 - RUNNING a RUNNABLE.
 - RUNNING a ZOMBIE.
 - RUNNING a UNUSED (TERMINATED).
 - SLEEPING a RUNNABLE.
 
 Más adelante nos explayaremos más profundamente sobre estos cambios de estados.
***

### **¿Qué es un *quantum*? ¿Dónde se define en el código? ¿Cuánto dura un *quantum* en `xv6-riscv`?**

Un quantum es un intervalo de tiempo que un proceso tiene para ejecutarse antes de que el scheduler pase al siguiente proceso. Pero el quantum no es un valor que no tenga importancia, de hecho, es muy importante ya que su tamaño puede influir en el rendimiento del sistema, afectando al tiempo de respuesta y de espera por ejemplo.

Un quantum muy corto (o chico) puede llevar a un alto overhead por tantos cambios de contexto, lo que generaría una disminución de la eficiencia del sistema en general. En cambio, si el quantum es largo (o grande), puede hacer que los procesos tengan que esperar mucho más tiempo para poder ejecutarse y se sientan atrasados. Lo mejor es buscar un quantum lo más equilibrado posible para mejorar la experiencia del usuario y optimizar lo mejor posible el sistema y su carga de trabajo.

En el código no encontramos al quantum con ese nombre en especifico, por lo tanto es más complicado ubicarlo. Sin embargo, lo encontramos en el directorio `so24lab3g17/kernel/start.c`, específicamente es la función `timerinit()`
la cual tiene como función pedir que se hagan interrupciones. Un quantum en xv6-riscv dura 10 ms en general, pero con esta implementación, vemos que el intervalo es 1000000 y si hacemos el calculo, el quantum es de 1 segundo o 1000 ms.

![timerinit](https://github.com/user-attachments/assets/d4aa9f21-ded4-4c72-8d34-e7c05841ecec)

***

### **¿En qué parte del código ocurre el cambio de contexto en `xv6-riscv`? ¿En qué funciones un proceso deja de ser ejecutado? ¿En qué funciones se elige el nuevo proceso a ejecutar?**


En xv6-RISCV, el cambio de contexto es cuando el S.O detiene la ejecución de un proceso para dar paso a la
ejecución de otro. Este cambio de contexto guarda toda la información del proceso que se detuvo.

El cambio de contexto ocurre en la función `swtch()`, que se encuentra en el archivo `kernel/swtch.S`, que está escrito en código ensamblador. El proceso del cambio de contexto ocurre en `scheduler()`, la podemos encontrar en el directorio `so24lab3g17/kernel/proc.c`. Dicha función selecciona el próximo proceso que debe ejecutarse de los que están en estado RUNNABLE y una vez encontrado uno se invoca el cambio de contexto.

Un proceso deja de ser ejecutado ya sea de forma voluntaria o involuntaria. La función `sched()` (en
`kernel/proc.c`)es una función que organiza el cambio de contexto en el sistema operativo. Dicha función se encarga de la planificación, realizar el cambio de contexto a otro proceso (mediante la función `swtch()`) y realiza comprobaciones para asegurar que el proceso actual no este en un estado incorrecto.

Una función cuando llama a `sched()` detiene la ejecución de un proceso guardando el contexto del proceso actual y luego con la llamada `swtch()` cede la ejecución a otro proceso.

Algunas funciones que llaman a sched son:

- yield() (en `kernel/proc.c`)
- sleep() (en `kernel/proc.c`)
- exit()  (en `kernel/proc.c`)

En las siguientes funciones también un proceso deja de ser ejecutado:

- wait()  (en `kernel/proc.c`)
- swtch() (en `kernel/swtch.S`)
- sched() (en `kernel/proc.c`)


El planificador elige un nuevo proceso a ejecutar en la función `scheduler()`, donde recorre la lista de procesos en busca de uno en estado RUNNABLE y realiza el cambio de contexto. En este caso `swtch()` facilita el cambio de contexto para ejecutar el proceso seleccionado.
***

### **¿El cambio de contexto consume tiempo de un *quantum*?**

El cambio de contexto no afecta directamente al tiempo de un quantum, es decir, el tiempo del quantum se mide únicamente mientras el proceso está ejecutando su código, y no incluye el tiempo al cambio de contexto.


***

## Segunda Parte


Antes de comenzar a responder a las preguntas correspondientes a los experimentos; detallaremos brevemente nuestras métricas, observaciones, anotaciones y consideraciones a tener en cuenta a la hora de analizar los datos recopilados.

En cuanto a nuestras métricas podemos decir que utilizamos la misma (en cierto sentido) para cpubench e iobench; solo que la de este último presenta una pequeña variación. Habiendo dicho esto nuestra métrica se basa en obtener un punto en común que sea fácilmente equiparable entre ambos procesos a ejecutar; para ello decidimos hacer una unidad de medición de cantidad de operaciones por tick.

Definida de la siguiente manera: *Total_Ops/Elapsed_Time*

Dónde Total_Ops se refiere a la cantidad total de operaciones que realiza un proceso (ya sea iobench o cpubench) durante su ejecución; ahora vale aclarar que dicha cantidad siempre es constante para dicho tipo de proceso; siendo **536832** para los *CPUs* y **1024** para los *IOs*.

Luego Elapsed_Time se refiere al tiempo que le llevó terminar a un proceso particular su ejecución, *medido en ticks*; donde recordemos que en general **un tick equivale a 10 ms**.

Ahora una vez que tuvimos las métricas solo quedaba analizar; pero debido a que tuvimos que considerar un pequeño inconveniente debimos modificar la métrica de los iobenchs quedándonos de la siguiente manera: *(Total_Opsx1000)/Elapsed_Time*

esto sucedió en parte debido a que al no haber flotantes en xv6 y ser los valores de Total_Ops pequeños en comparación a los del cpubench; esto nos da valores de una sola cifra. Para solucionar dicho caso tuvimos que multiplicar el valor antes de ser dividido y lo hicimos arbitrariamente por 1000 debido a que con esto conseguimos una precisión de 4 decimales. Por esto **de ahora en adelante siempre que se vea un valor de resultado relacionado a iobench se debe recordar que este es en realidad 1000 veces más pequeño.**

  

Una vez comprendido lo anterior pasaremos a explicar cómo leer los resultados de nuestras tablas: las cuales se encuentran en el siguiente link para su observación en caso de que se presente alguna duda o incongruencia con los datos aqui especificados además de presentar informacion extra:
https://docs.google.com/spreadsheets/d/18brDhybx47M2KcV9tvuRznninte5-okGpbE052r3r_4/edit?usp=sharing


Todas nuestras tablas son de la siguiente forma y sus correspondientes conjunciones posibles:
| Pid | Metrica                  | Resultado  | S.T  | T.T  |
|-----|--------------------------|------------|------|------|
| a   | (total_ops * 1000) / E.T | 7262       | 796  | 141  |
| a   | (total_ops * 1000) / E.T | 6965       | 937  | 147  |
| a   | (total_ops * 1000) / E.T | 7366       | 1084 | 139  |
| a   | (total_ops * 1000) / E.T | 7420       | 1224 | 138  |
| a   | (total_ops * 1000) / E.T | 7474       | 1363 | 137  |
| a   | (total_ops * 1000) / E.T | 7366       | 1500 | 139  |
| a   | (total_ops * 1000) / E.T | 7529       | 1639 | 136  |

| Pid | Metrica       | Resultado  | S.T  | T.T  |
|-----|---------------|------------|------|------|
| 1   | total_ops/E.T | 16776      | 845  | 32   |
| 1   | total_ops/E.T | 16776      | 877  | 32   |
| 1   | total_ops/E.T | 17317      | 909  | 31   |
| 1   | total_ops/E.T | 17317      | 941  | 31   |
| 1   | total_ops/E.T | 17317      | 973  | 31   |
| 1   | total_ops/E.T | 17317      | 1005 | 31   |



Como podemos apreciar en todas ellas tenemos los siguientes parámetros:

- Pid: corresponde a la identificación del proceso; para los cpubench se utilizaran números y para los iobench letras con orden alfabetico; facilitando la rápida y precisa comprensión del tipo de proceso que se está viendo.
- Métrica: como indica su nombre esta tendrá la correspondiente métrica del proceso en ejecución.
- Resultado: Esta indica el valor resultante de aplicar la anterior métrica; reiteramos *el resultado en los tipo iobench es 1000 veces menor al aquí expresado*.
- S.T: denota la abreviación de *Starting Time* y sirve para ver que proceso llegó primero a la process table y en qué orden los ejecuto el scheduler.
- T.T: representa el *Ticks Time* es decir el tiempo en ticks que le llevó a los procesos en ejecutarse.
  

Ahora respecto a los promedios utilizados en nuestros análisis; debemos aclarar que *todos los comandos se ejecutaron* **3 veces** para su posterior promediación. Esto resultó en los siguiente tipos de promedios:

**Promedios Simples**

- RF: denota *Resultado Final* y se refiere al promedio de todos los resultados parciales obtenidos (siempre que estas consistieran en las de un único tipo; es decir solo se ejecutaron todos cpu/io benchs).
- TTF: en este caso se refiere al *Tiempo Total Final* es decir es el cálculo estimativo que nos permite decir cuánto tiempo hubiera tardado el procesador en obtener un resultado similar al que nos arrojó anteriormente el RF. Este está medido en *ticks* y para calcularlos difiere en dos casos:
1. Cpubench: es producto de la división del número total de operaciones (536832) sobre el resultado de RF.
2. Iobench: se deduce del número total de operaciones (1024) multiplicado por 1000 dividido el total resultante de RF.

**Promedios Compuestos**

- RCF: denota *Resultado de Cpus Final* y es un RF pero solo correspondiente a los procesos de tipo cpubench que aparezcan en dicha tabla.
- TTCF: refiriendose al *Tiempo Total de Cpus Final* es un TTF con el resultado del RCF.
- RFI: nos da el *Resultado Final de Iobench* y es igual al RCF pero para su tipo.
- TTIF: es el *Tiempo Total de Iobench Final* y es el TTF de los RIF.
- RTF: nos denota el *Resultado Total Final* y es el promedio de resultante del RCF y el RIF.
- TTFC: o mejor dicho el *Tiempo Total Final Compuesto* es el promedio de los TTIF y el TTCF.

**Debe observarse que los resultados pertinentes a los IOBENCHS no fueron divididos por 1000 para mantener una fácil cohesión entre los resultados en general y facilitar su cálculo promedial en sí; ya que los resultados de estos eran tan ínfimos que no impactan significativamente en dichos promedios.**

***

### **Describa los parámetros de los programas cpubench e iobench para este experimento (o sea, los define al principio y el valor de N. Tener en cuenta que podrían cambiar en experimentos futuros, pero que si lo hacen los resultados ya no serán comparables).**

**cpubench.c:**

  - `CPU_MATRIX_SIZE`: Define el tamaño de las matrices cuadradas (Fijo en 128)
     
- `CPU_EXPERIMENT_LEN`: Cantidad de veces que se realiza la multiplicación de matrices (fijo en 256)

- `N`: Representa la cantidad de veces que se realiza el ciclo de medición en cpubench.

**iobench.c:**

- `IO_OPSIZE`: Tamaño de cada operación de entrada/salida (fijo en 64)

- `IO_EXPERIMENT_LEN`: Número de operaciones de escritura y lectura en cada ciclo (fijo en 512)

- `N`: Representa la cantidad de veces que se realiza el ciclo de medición en iobench

***

### ¿Los procesos se ejecutan en paralelo? ¿En promedio, qué proceso o procesos se ejecutan primero? Hacer una observación cualitativa.

En nuestras mediciones observamos que si, los procesos se ejecutan en paralelo. Por ejemplo, al ejecutar el comando `iobench 7 &; iobench 7 &; iobench 7 &`, hay 2 procesos IO que se empiezan a ejecutarse en
el mismo tick (es decir, S.T: 2677), además, los procesos no se ejecutan de forma secuencial. 
En promedio los procesos cpubound se ejecutan primero ya que su ejecución es mucho más rápida. Por ejemplo, si ejecutamos 3 cpubench y 1 iobench, la ejecución del iobench al tardar más que un quantum,
recibirá la interrupción e irá a los demás procesos cpu que se terminan de ejecutar más rápido, por lo tanto terminaran los 3 cpubench antes que el iobench.

***

### ¿Cambia el rendimiento de los procesos iobound con respecto a la cantidad y tipo de procesos que se estén ejecutando en paralelo? ¿Por qué?

Si, en las mediciones realizadas el rendimiento de los procesos iobound cambia con respecto a la cantidad y tipo de procesos. Al ejecutar un proceso iobound el rendimiento es menor que al ejecutar 3
procesos iobound. Al ejecutarse un iobound junto a procesos cpubound el rendimiento disminuye levemente comparado a un solo proceso iobound y esto se debe porque cpubound utiliza recursos del cpu mientras
que el iobound utilizan las operaciones de entrada/salida por ende no se van tan afectado el rendimiento. En conclusión la ejecución de varios procesos iobound en paralelo mejora el rendimiento total
comparado con la ejecución de un solo proceso iobound y se ve afectado levemente con la ejecución de procesos cpubound en paralelo.

***

### ¿Cambia el rendimiento de los procesos cpubound con respecto a la cantidad y tipo de procesos que se estén ejecutando en paralelo? ¿Por qué?

Si, logramos notar que el rendimiento de los procesos cpubound cambia dependiendo de la cantidad de procesos. Por ejemplo, al ejecutar 3 procesos cpubound el rendimiento decae mucho al compararlo con la
ejecución de un solo proceso cpubound, pero si ejecutamos un cpubound junto a procesos iobound, el rendimiento casi que no disminuye.
Esto ocurre porque al ser tan rápidos los procesos cpubound, compiten por los recursos de la cpu, entonces se pierde la eficiencia al ser ejecutados en UN solo cpu.

***

### ¿Es adecuado comparar la cantidad de operaciones de cpu con la cantidad de operaciones iobound?

No, no es adecuado ya que nuestras métricas de cpubench e iobench tienen una ligera variación, además cada tipo de operación representa una carga distinta para el sistema y utiliza distintos recursos, por
ende, no es adecuado comparar la cantidad de operaciones de cpubound con la cantidad de operaciones iobound.

