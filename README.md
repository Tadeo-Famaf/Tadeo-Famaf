# INFORME XV6-RISCV

### **¿Qué política de planificación utiliza `xv6-riscv` para elegir el próximo proceso a ejecutarse?**

Xv6-riscv usa *RR (Round Robin)* como política de planificación. 

**Así funciona:**





![Captura de pantalla](~/Screenshots/scheduler.png)


![scheduler](https://github.com/user-attachments/assets/085626c1-4c3d-4007-9036-8e035b6a4758)








***

### **¿Qué es un *quantum*? ¿Dónde se define en el código? ¿Cuánto dura un *quantum* en `xv6-riscv`?**

Un quantum como ya vimos, es un intervalo de tiempo que un proceso tiene para ejecutarse antes de que el scheduler pase al siguiente proceso. Pero el quantum no es un valor que no tenga importante, de hecho, es muy importante ya que su tamaño puede influir en el rendimiento del sistema, afectando al tiempo de respuesta y de espera por ejemplo. Un quantum muy corto (o chico) puede llevar a un alto overhead por tantos cambios de contexto, lo que generaria una disminución de la eficencia del sistema en general. En cambio, si el quantum es largo (o grande), puede hacer que los procesos tengan que esperar mucho más tiempo para poder ejecutarse y se sientan atrasados. Lo mejor es buscar un quantum lo más equilibrado posible para mejorar la expericneic adel usuario y optimizar lo mejor posible el sistema y su carga de trabajo.


