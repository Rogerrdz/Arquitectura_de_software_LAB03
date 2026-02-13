# Arquitectura_de_software_LAB03
### Author : Roger Rodriguez

---

# Actividades del laboratorio

## Parte I — (Antes de terminar la clase) `wait/notify`: Productor/Consumidor
1. Ejecuta el programa de productor/consumidor y monitorea CPU con **jVisualVM**. 

- Primero ejecutamos el programa con Busy - Wait (alto CPU) para monitoriarlo usando :

```bash
mvn -q -DskipTests exec:java -Dexec.mainClass=edu.eci.arsw.pc.PCApp \
  -Dmode=spin -Dproducers=1 -Dconsumers=1 -Dcapacity=8 -DprodDelayMs=50 -DconsDelayMs=1 -DdurationSec=30
``` 

- Nos dirigimos a jvisualVM vamos al apartado de Hilos asi como el comportamiento:

![Resumen](/assets/Jvisual_vista_general_primera_ejecucion%20.png)

- Hilos :

![Hilos Busy Wait](/assets/Jvisual_Hilos_busy_wait_alto_cpu.png)

- Monitoreo :

![Monitoreo busy wait](/assets/Jvisual_Monitoreo_busy_wait_alto_cpu%20.png)

- Ahora ejecutamos el programa con monitores (uso eficiente de CPU)

```bash
mvn -q -DskipTests exec:java -Dexec.mainClass=edu.eci.arsw.pc.PCApp \
  -Dmode=monitor -Dproducers=1 -Dconsumers=1 -Dcapacity=8 -DprodDelayMs=50 -DconsDelayMs=1 -DdurationSec=30
```
- Nos dirigimos a jvisualVM vamos al apartado de Hilos asi como el comportamiento :
  
![Resumen](/assets/Jvisual_vista_general_segunda_ejecucion%20.png)

- Hilos :

![Hilos Busy Wait](/assets/Jvisual_Hilos_busy_wait_alto_cpu.png)

- Monitoreo :

![Monitoreo busy wait](/assets/Jvisual_Monitoreo_monitores_optimo_cpu.png)

- ¿Por qué el consumo alto? ¿Qué clase lo causa? :

Al ejecutar en modo spin, el consumo de CPU es alto porque la clase BusySpinQueue utiliza busy-wait que es espera activa . Tanto el metodo put() como take() tienen un bucle while(true) que no cede el control del procesador; simplemente pregunta una y otra vez si hay espacio o si hay elementos sin pausarseademas el hilo consumidor nunca se duerme , continua consumiendo cpu sin hacer ningun trabajo.

2. Ajusta la implementación para **usar CPU eficientemente** cuando el **productor es lento** y el **consumidor es rápido**. Valida de nuevo con VisualVM.  

- Para esto nos fijamos que la solucion se encuentra en Boundeduffer ya que este actua como la bandeja de produccion que se consume ; utilizando 'syncronized' , 'wait()' y 'notifyAll()' , cuando el consumidor llama a 'take()' y la cola se encuentra vacia, ejecuta 'this.wait()' lo que pone el hilo a dormir hasta que el productor ponga algo llamando a 'notifyAll()':

- El metodo 'take()' con la implementacion: 

```java
public T take() throws InterruptedException {
    synchronized (this) {
        while (q.isEmpty()) {
            this.wait(); // Si esta vacio el hilo se duerme 
        }
        T v = q.removeFirst();
        this.notifyAll(); //se  despierta al productor si estaba esperando
        return v;
    }
}
```
- Verificamos el comportamiento usando visualVM :

![Monitoreo despues del arreglo](/assets/Jvisual_Monitoreo_monitores_optimo_cpu_arreglado.png)

3. Ahora **productor rápido** y **consumidor lento** con **límite de stock** (cola acotada): garantiza que el límite se respete **sin espera activa** y valida CPU con un stock pequeño.

- Para esto nos fijamos en que cuando el productor es rapido , el consumidor es lento y se tiene limitado el stock la cola se llena rapidamente y esto se encuentra implicado en el metodo 'put()' de BoundedBuffer la cual usa un ciclo while que duerme el hilo respetando el limite :

- El metodo 'put()' con la implementacion :

```java
public void put(T item) throws InterruptedException {
    synchronized (this) {
      while (q.size() == capacity) {
        this.wait(); //se espera hasta que haya espacio
      }
      q.addLast(item);
      this.notifyAll(); //se despierta a los consumidores
    }
  }
```
- Verificamos el comportamiento usando visualVM :

![Monitoreo despues del arreglo](/assets/Jvisual_Monitoreo_monitores_optimo_cpu_arreglado_put.png)

---

### Escenario 2 : 
Productor rápido / Consumidor lento con límite de stock → productor debe esperar sin CPU cuando la cola esté llena (capacidad pequeña, ej. 4 u 8).

- Con Spin (CPU alto) :
  
```java
mvn -q exec:java -Dmode=spin -Dproducers=1 -Dconsumers=1 -Dcapacity=4 -DprodDelayMs=1 -DconsDelayMs=200 -DdurationSec=30
```
- Hilos :
  
![Hilos 4](/assets/Jvisual_Hilos_spin_esc_2.png)

- Monitoreo : 

![Monitoreo](/assets/Jvisual_Monitoreo_spin_esc_2.png)

- Con monitores (CPU optimo):
```java
mvn -q exec:java -Dmode=monitor -Dproducers=1 -Dconsumers=1 -Dcapacity=4 -DprodDelayMs=1 -DconsDelayMs=200 -DdurationSec=30
```
- Hilos :
  
![Hilos 4](/assets/Jvisual_Hilos_monitores_esc_2.png)

- Monitoreo : 

![Monitoreo](/assets/Jvisual_Monitoreo_monitores_esc_2.png)

---

## Parte II — (Antes de terminar la clase) Búsqueda distribuida y condición de parada
Reescribe el **buscador de listas negras** para que la búsqueda **se detenga tan pronto** el conjunto de hilos detecte el número de ocurrencias que definen si el host es confiable o no (`BLACK_LIST_ALARM_COUNT`). Debe:
- **Finalizar anticipadamente** (no recorrer servidores restantes) y **retornar** el resultado.  
- Garantizar **ausencia de condiciones de carrera** sobre el contador compartido.

### Desarrollo :

1. HostBlackListValidator

- Agregamos un contador que sea compartido asi como su metodo sincronizado para incrementarlo:

```java
    
    private int occurrencesCount = 0;
    
    public synchronized int increaseOccurrences() {
    occurrencesCount++;
    if (occurrencesCount >= BLACK_LIST_ALARM_COUNT) {
        stop = true;
    }
    return occurrencesCount;
}
```

- Asignamos valores a las variables en el metodo de 'checkHost()' asi como la lista sincronizada, tambien le pasamos el validador al hilo :

```java
    stop = false;
    occurrencesCount = 0;
    List<Integer> blackListOcurrences = java.util.Collections.synchronizedList(new LinkedList<>());
    SearchBlackListThread t = new SearchBlackListThread(start, end, ipaddress, blackListOcurrences, this);

```
2. SearchBlackListThread

- Agregamos el validador y modificamos el constructor:

```java
    private HostBlackListsValidator validator;
    public SearchBlackListThread(int start, int end, String ipaddress,
            List<Integer> blackListOcurrences,
            HostBlackListsValidator validator) {

        this.start = start;
        this.end = end;
        this.ipaddress = ipaddress;
        this.blackListOcurrences = blackListOcurrences;
        this.validator = validator;
    }
```

- Modificamos el metodod run 'run()':

```java
    @Override
    public void run() {

        HostBlacklistsDataSourceFacade skds =
                HostBlacklistsDataSourceFacade.getInstance();

        for (int i = start; i < end && !validator.Stop(); i++) {

            if (skds.isInBlackListServer(i, ipaddress)) {

                blackListOcurrences.add(i);

                validator.increaseOccurrences();
            }
        }
    }
```

Cada hilo estaba creando su propia instancia de HostBlackListsValidator, lo que impedia que compartieran el mismo contador y la misma bandera de parada, no existia coordinacion entre ellos y la busqueda no podia detenerse antes. Para que funcionara correctamente, eliminamos la creacion de un nuevo validator dentro del hilo, pasamos la misma instancia compartida desde checkHost, corregimos las variables y aseguramos que el contador y la bandera stop fueran compartidos y sincronizados, permitiendo así detener la búsqueda cuando se alcanza BLACK_LIST_ALARM_COUNT sin condiciones de carrera.

---

