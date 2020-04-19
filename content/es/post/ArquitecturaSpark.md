---
title:  "Arquitectura de Spark"
date:   2019-07-13 13:14:54
---
## **Arquitectura de Spark**

La clave del asombroso performance de Spark es el paralelismo:

* El escalado vertical -> está limitado por una cantidad finita de RAM, los Threads disponibles en una máquina o la velocidad del CPU.

* El escalado horizontal -> se pueden anadir nuevos nodos al cluster de forma casi indefinida, aumentando el rendimiento de las aplicaciones.

Spark paraleliza a dos niveles:

* El primer nivel de paralelización es el ejecutor, una JVM que corre un nodo. Típicamente una instancia por nodo.

* El segundo nivel es el "Slot" - Un número que viene determinado por el número de cores y CPU's de cada nodo.

### **Driver**  

Es el controlador de la ejecución de una Aplicación de Spark y mantiene el estado del cluster. Interactua con el Gestor del Clúster de cara a obtener recursos físicos y lanzar los Ejecutores.  

No es más que un proceso Java (SparkContext) que corre en una máquina física que es responsable de mantener el estado de la aplicación.


### **Executor**

Los ejecutores de Spark son los procesos que realizan las tareas asignadas por el Driver. Estos ejecutores tienen una responsabilidad clave: Coger la tarea asignada por el driver, ejecutarla, reportar su estado (Exito o Fracaso) y los resultados. Cada aplicación de Spark tiene sus propios procesos de ejecución.  

Cada ejecutor tiene un número de Slots en los que puede paralelizar la tareas.

### **Jobs**

Cada acción paralelizada se conoce como un Job. El resultado de cada Job se envía al Driver. Dependiendo del trabajo requerido, puede que se requieran múltiples Jobs.  

Computar un Job es equivalente a trajar sobre una partición de un RDD la acción sobre la que se ha ejecutado. El número de particiones de un Job depende del tipo de Stage (ResultStage o ShuffleMapStage)

### **Stages**

Un stage es una unidad física de ejecución. Es un paso en el plan de ejecución físico.  

Los Stages son el resultado de dividir un Job.  

Un Stage es un conjunto de Tasks paralelas -> Un task por partición de un RDD que computa el resultado parcial de un Job.

### **Tasks**

Un Task es una abstracción de la más pequena unidad de ejecución individual que se puede ejecutar en una sola partición.  


### **Partitions**

Spark gestiona los datos usando particiones que le ayudan a paralelizar los datos distribuidos con un mínimo tráfico de red.  

Por defecto una parcition en Spark es creada por cada particion en HDFS (64 MB por defecto).  

Por lo general, muchas particiones pequenas ayudan a que el trabajo sea distribuido entre más workers, pero pocas particiones grandes permiten que el trabajo sea realizado en grandes lotes, lo que puede significar un mejor rendimiento.  

Spark solo puede correr una tarea concurrente por cada aparticion. Por lo que si hay 50 cores, solo se podrán ejecutar 50 Tasks a la vez.  

El tamano máximo de una particion está delimitado por el tamano de la memoria del ejecutor.

### **Shuffling**

Shuffling es el proceso de redistribuir datos entre las distintas particiones que puede o no causar el movimiento de datos entre los distintos procesos de la JVM.  

Es el proceso de transferencia de datos entre Stages.  

Hay que evitarlo a toda costa, ya que es una operación muy costosa. Por defecto, el Shuffling no cambia el número de particiones, pero si su contenido.  


#### **(Más detalle)**

Para llevar a cabo un Shuffle, Spark necesita:  

* Convertir los datos a UnsafeRow - Tungsten Binary Format.
* Escribir los datos a disco en el nodo local - En este punto habría un slot libre para la siguiente tarea.
* Enviar los datos a otros ejecutores.
* El driver decide qué ejecutor recibe qué dato.
* Copiar los datos de nuevo a la RAM en el nuevo ejecutor.


#### **(Side Note)**
UnsafeRow: es el almacenamiento In Memory que Spark usa para estructuras de datos (DataFrames y DataSets).
* Los valores de las columnas son codificados usando Encoders.
* Misma compacidad que la Serializacion Java, pero más rápido.
* Es posible escribir Encoders personalizados.
* Es más eficiente, ya que spark no tiene que deserializar los datos.


### **Stages (Parte 2)**
Cuando se hace un Shuffle de los datos, se crea una Stage Boundary.  

En un Job tal que asi:  

    Read -> Filter -> GroupBy -> Select -> Filter -> Write

Se crearian dos Stages:

    Read -> Filter -> GroupBy -> Shuffle Write
    Shuffle Read -> Group By -> Select -> Filter -> Write

Todas las particiones deben completar el Stage 1 antes de continuar con el 2.
