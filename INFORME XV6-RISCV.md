# INFORME XV6-RISCV

## Primera Parte

### **¿Qué política de planificación utiliza `xv6-riscv` para elegir el próximo proceso a ejecutarse?**

Xv6-riscv usa *RR (Round Robin)* como política de planificación. La implementación la podemos encontrar en el directorio `so24lab3g17/kernel/proc.c`, en específico en la función `scheduler()`.

**Así funciona:**

![scheduler](https://github.com/user-attachments/assets/085626c1-4c3d-4007-9036-8e035b6a4758)


Podemos observar que la función scheduler contiene 2 loops, el primer loop (`for(;;)`) es infinito, esto para buscar continuamente procesos que puedan ejecutarse, luego dentro de este loop se habilitan las interrupciones con `intr_on()` para evitar un bloqueo si los procesos están en espera.

En el segundo loop (`for(p = proc; p < &proc[NPROC]; p++)`) se itera sobre la lista de procesos, después se adquiere el lock (o bloqueo) del proceso p (`acquire(&p->lock)`) y a continuación se verifica si
el proceso p se encuentra en estado `RUNNABLE`, lo que significa que está listo para ejecutarse.

Si el proceso está en RUNNABLE, se procede a cambiarle el estado a `RUNNING` para indicar que este nuevo proceso p está en ejecución, luego se guarda el proceso actual (p) en el contexto de la CPU y se llama a la función `swtch`
para realizar un cambio de contexto entre el proceso que se **estaba** ejecutando y el **nuevo** proceso p y así se ejecuta por fin el proceso p.

Posteriormente, se libera el lock (`release(&p->lock)`).

Ahora, qué ocurre si no hay ningún proceso en estado RUNNABLE (es decir `if(found == 0)`)? bueno, el sistema esperará en un estado de bajo consumo (`asm volatile("wfi")`) y de inactividad en ese núcleo hasta que se produzca algún tipo de interrupción.



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
la cual tiene como función pedir que se hagan interrupciones. Un quantum en xv6-riscv dura 10 ms.

![timerinit](https://github.com/user-attachments/assets/92208d04-6f24-4f61-8dfc-684886190890)
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
