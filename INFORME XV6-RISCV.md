# INFORME XV6-RISCV

## Primera Parte

### **¿Qué política de planificación utiliza `xv6-riscv` para elegir el próximo proceso a ejecutarse?**

Xv6-riscv usa *RR (Round Robin)* como política de planificación. La implementación la podemos encontrar en el directorio `so24lab3g17/kernel/proc.c`, en especifico en la función `scheduler()`.

**Así funciona:**

![scheduler](https://github.com/user-attachments/assets/085626c1-4c3d-4007-9036-8e035b6a4758)


Podemos observar que la función scheduler contiene 2 loops, el primer loop (`for(;;)`) es infinito, esto para buscar continuamente procesos que puedan ejecutarse, luego dentro de este loop se habilitan las interrupciones con `intr_on()` para evitar un bloqueo si los procesos estan en espera.

En el segundo loop (`for(p = proc; p < &proc[NPROC]; p++)`) se itera sobre la lista de procesos, después se adquiere el lock (o bloqueo) del proceso p (`acquire(&p->lock)`) y a continuación se verifica si
el proceso p se encuentra en estado `RUNNABLE`, lo que significa que está listo para ejecutarse.

Si el proceso está en RUNNABLE, se procede a cambiarle el estado a `RUNNING` para indicar que este nuevo proceso p está en ejecución, luego se guarda el proceso actual (p) en el contexto de la CPU y se llama a la función `swtch`
para realizar un cambio de contexto entre el proceso que se **estaba** ejecutando y el **nuevo** proceso p y asi se ejecuta por fin el proceso p.

Posteriormente, se libera el lock (`release(&p->lock)`).

Ahora, que ocurre si no hay ningún proceso en estado RUNNABLE (es decir `found == 0`)? bueno, el sistema esperará en un estado de bajo consumo (`asm volatile("wfi")`) y de inactividad en ese núcleo hasta que se produzca algún tipo de interrupción.



***

### **¿Qué es un *quantum*? ¿Dónde se define en el código? ¿Cuánto dura un *quantum* en `xv6-riscv`?**

Un quantum es un intervalo de tiempo que un proceso tiene para ejecutarse antes de que el scheduler pase al siguiente proceso. Pero el quantum no es un valor que no tenga importancia, de hecho, es muy importante ya que su tamaño puede influir en el rendimiento del sistema, afectando al tiempo de respuesta y de espera por ejemplo. Un quantum muy corto (o chico) puede llevar a un alto overhead por tantos `cambios de contexto`, lo que generaria una disminución de la `eficencia` del sistema en general. En cambio, si el quantum es largo (o grande), puede hacer que los procesos tengan que esperar `mucho más` tiempo para poder ejecutarse y se sientan `atrasados`. Lo mejor es buscar un quantum lo más equilibrado posible para mejorar la expericneic adel usuario y optimizar lo mejor posible el sistema y su carga de trabajo.

En el código no encontramos al quantum con ese nombre en especifico, por lo tanto es más complicado ubicarlo. Sin embargo, lo encontramos en el directorio `so24lab3g17/kernel/start.c`, especificamente es la función `timerinit()`
la cual tiene como función pedir que se hagan interrupciones después de tal tiempo. Un quantum en xv6-riscv dura 10 ms.

![timerinit](https://github.com/user-attachments/assets/92208d04-6f24-4f61-8dfc-684886190890)



