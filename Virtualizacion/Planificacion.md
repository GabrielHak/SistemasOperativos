# Parte I &rarr; Virtualizacion

Temas:

* [Procesos](./Procesos.md)
* [API de procesos](./API-de-procesos.md)
* [Ejecucion directa limitada](./Ejecucion-directa.md)
* [Planificacion](#introduccion-planificacion-scheduling): &larr; Usted esta aqui

  * [Suposiciones de carga de trabajo](#suposiciones-de-carga-de-trabajo)
  * [Metricas de planificacion](#metricas-de-planificacion)
  * [First In, First Out (FIFO)](#first-in-first-out-fifo)
  * [Shortest Job First (SJF)](#shortest-job-first-sjf)
  * [Shortest Time-to-Completion First (STCF)](#shortest-time-to-completion-first-stcf)
  * [Una nueva metrica: Tiempo de respuesta (Respose Time)](#una-nueva-metrica-tiempo-de-respuesta-response-time)
  * [Round Robin](#round-robin)
  * [Incorporando I/O](#incorporando-io)

* [Planificacion multinivel](./Planificador-multinivel.md)
* [La abstraccion del espacio de direcciones](./Espacio-direcciones.md)
* [API de memoria](./API-memoria.md)
* [El mecanismo de traduccion de direcciones](./Traduccion-direcciones.md)
* [Segmentacion](./Segmentacion.md)
* [Administracion de espacio libre](./Espacio-libre.md)
* [Paginacion](./Paginacion.md)
* [TLBs](./TLBs.md)
* [Paginacion multinivel](./Paginacion-Multinivel.md)

Bibliografia: [OSTEP Cap - 7 Scheduling: Introduction](https://pages.cs.wisc.edu/~remzi/OSTEP/cpu-sched.pdf)

## Introduccion: Planificacion (Scheduling)

&emsp;Por ahora los mecanismos de bajo nivel (como el cambio de contexto) deben estar claros. Sin embargo, todavia tenemos que entender las **politicas** de alto nivel que emplea el OS. Ahora vamos a hacer eso, presentar un par de **politicas de planificacion** (a veces llamadas **disciplinas**).</br>
&emsp;El origen de la planificacion, de hecho, precede a los sistemas computacionales; los primeros enfoques fueron tomados del campo de gestion de operaciones y aplicadas a la computacion</br>

### Suposiciones de carga de trabajo

&emsp;Antes de entrar en el rango de las posibles politicas, hagamos algunas suposiciones simplificadoras sobre los procesos en ejecucion del sistema, a veces llamado **la carga de trabajo (workload)**. Determinar la carga de trabajo es una parte critica en la construccion de politicas, y mientras mas sepas sobre la carga de trabajo, estaran mejor desarrolladas tus politicas.</br>
&emsp;Las suposiciones de carga de trabajo que haremos son mayormente irrealistas, pero por ahora esta bien, porque las relajaremos a medida que avancemos, y eventualmente desarrollaremos lo que llamaremos... (*dramatic pause*) ... una **disciplina de planificacion completamente funcional**</br>
&emsp;Haremos las siguientes suposiciones sobre los procesos que se estan ejecutando en el sistema.</br>

  1. Cada proceso se ejecuta por la misma cantidad de tiempo
  2. Todos los procesos llegan al mismo tiempo
  3. Una vez empezado, se ejecuta hasta completarse
  4. Todos los procesos solo usan la CPU (es decir, no hacen I/O)
  5. El tiempo de ejecucion de cada proceso es conocido

### Metricas de planificacion

&emsp;Mas alla de asusmir la carga de trabajo, necesitamos una cosa mas para ser capaces de comparar diferentes politicas de planificacion: una **metrica de planificacion**. Una metrica es solo algo que usamos para *medir* algo, y hay muchas metricas diferentes que tienen sentida en la planificacion.</br>
&emsp;Sin embargo, por ahora, vamos a simplificarnos la vida, tomando una solo metrica: **turnaround time (tiempo de entrega?)**. El tiempo de entrega de un proceso esta definido por el tiempo en el que el proceso finalizo menos el tiempo en el que el proceso llego al sistema. Mas formalmente, el tiempo de entrega is:</br>

$$T_{turnaround} = T_{completion} - T_{arrival}$$

&emsp;Y como asumimos que todos los procesos llegan al mismo tiempo, por ahora, el tiempo de llegada es igual a 0</br>

$$T_{arrival} = 0$$
$$T_{turnaround} = T_{completion} - 0$$
$$T_{turnaround} = T_{completion}$$

&emsp;Como habras notado, el tiempo de entrega es una metrica de desempeño, en el cual estara nuestro foco de atencion. Otra metrica interesante es **equidad (fairness)**. Desempeño y equidad a menudo estan en conlifcto, en desacuerdo, en la planificacion; un planificador (scheduler), por ejemplo, podria optimizar el desempeño, pero con el costo de evitar que otros procesos se ejecuten, disminuyendo la equidad. Este problema nos muestra que la vida no siempre es perfecta.(filosofando ando)</br>

### First In, First Out (FIFO)

&emsp;El algoritmo mas basico que podemos implementar es conocido como planificacion **First In, First Out (FIFO)** o algunas veces **First Comes, Frist Served (FCFS)**</br>
&emsp;FIFO tiene muchas propiedades positvas: claramente es simple, y por lo tanto facil de implementar. Y, dadas nuestras suposisiones, funciona bastante bien.</br>
&emsp;Hagamos un ejemplo rapido. Imaginemos que tres procesos llegan al sistema, A, B, y C, aproximadamente al mismo tiempo (t arrival = 0).Dado que FIFO ubica a algun proceso primero, asumamos que A llego apenas antes que B, y que B llego apenas antes que C. Asumamos tambien que cada proceso se ejecuta por 10 segundos. Cual sera el **turnnaround time** promedio para estos procesos?</br>

![FIFO simple example](../Imagenes/FIFO-simple-example.png)

&emsp;De la imagen anterior podemos ver que A termina al 10, B al 20, y C al 30. Por lo tanto, el tiempo de entrega promedio para los 3 procesos es simple:</br>

$$\frac{10+20+30}{3} = 20$$
&emsp;Ahora suavicemos una de nuestras suposiciones. En particular, suavicemos la suposicion 1, y por lo tanto no asumiremos mas que los procesos se ejecutan la misma cantidad de tiempo. Como funciona FIFO ahora? Que tipo de carga de trabajo podrias construir para que FIFO funcione mal?</br>
&emsp;Veamos ahora como procesos de diferente longitud pueden llevar problemas a la planificacion FIFO. En particular, de nuevo vamos a asumir 3 procesos (A, B, y C), pero esta vez A se ejecuta 100s y C y B 10s cada una.</br>

![Why FIFO Is Not That Great](../Imagenes/FIFO-not-great.png)

&emsp;Como puedes ver en la imagen, el proceso A se ejecuta primero los 100s completos antes de que B y C siquiera tengan oportunidad de ejecutarse. Por lo tanto, el tiempo de entrega promedio es alto:</br>

$$\frac{100+110+120}{3} = 110$$

&emsp;Generalmente se refiere a este problema como **efecto convoy (convoy effect)**, donde un numero de potenciales consumidores de un recurso relativamente cortos quedan en colados detras de un consumidor de recursos de alto peso.</br>
&emsp;Entonces, que deberiamos hacer? Como podriamos desarrolar un mejor algoritmo para tratar con nuestra nueva realidad de procesos que se ejecutan por diferentes cantidades de tiempo.</br>

### Shortest Job First (SJF)

&emsp;Resulta que un enfoque muy simple resuelve este problema; de hecho, es una idea robada de investigacion de operaciones y aplicada a la planificacion de trabajos en los sistemas computacionales. Esta nueva disciplina de planificacion es conocida como **Shortest Job First (SJF)**, y el nombre es facil de recordar porque describe la politica casi por completo: ejecuta el proceso mas corto primero, y luego el siguiente mas corto, y asi sucesivamente.</br>

![SJF simple example](../Imagenes/SJF-simple-example.png)

&emsp;Tomemos el mismo ejemplo de antes pero esta vez con SJF como nuestra politica de planificacion. La imagen anterios muestra los resultados de ejecutar A, B, y C. El diagrama muestra porque SJF se desempeña mucho mejor con respecto al tiempo de entrega promedio. Simplemente ejecutando B y C antes que A, SJF reduce el tiempo promedio de 110 a 50 segundo, menos de la mitad del tiempo.</br>
&emsp;Por lo tanto llegamos a un buen enfoque al planificar con SJF, pero nuestras suposiciones todavia son irrealistas. Vamos a suavizar otra. En particular, la suposicion 2, y ahora asumimos que los procesos pueden llegar en cualquier comento, y no todos a la vez. A que problemas conduce esto?</br>

&emsp;Usaremos un ejemplo similiar, pero esta vez asumiremos que A llega en t = 0 y necsita ejecutarse por 100s, mientras que B y C llegan en t = 10 y cada uno necesita ejecutarse 10s. Con SJF puto, obtendremos una planificacion como la siguiente.</br>

![SJF With Late Arrivals From B And C](../Imagenes/SJF-late-arrival.png)

&emsp;Como se puede ver en la imagen, a pesar de que B y C llegaron poco despues que A, ellos son forzados a esperar a que A termine su ejecucion, y por lo tanto sufre el mismo efecto convoy. El tiempo de entrega promedio para estos tres procesos es de 103.33. Que puede hacer el planificador?</br>

$$\frac{100 + (110 - 10) + (120 - 10)}{3} = 103.33$$

### Shortest Time-to-Completion First (STCF)

&emsp;Para abordar esta preocupacion, necesitamos suavizar la suposicion 3 (los procesos se ejecutan hasta terminarse). Tambien necsitamos algo de maquinaria en el planificador. El planificador puede hacer algo cuando B y C llegan: puede **remplazar (preempt)** y decidir ejecutar otro proceso, y quizas continuar ejecutando A despues. SJF por nuestra definicion es un planificador sin reemplazos, y por lo tanto sufre de los problemos descritos arriba.</br>
&emsp;Afortunadamente, hay un planificador que hace exactamente eso, agregar reemplazos a SJF, y es conocido como planificador **Shortest Time-to-Completion First (STCF)** o **Preemtive Shortest Job First(PSJF)**. Cuando entra un nuevo proceso al sistema, el planificador STCF determina a cual de los procesos que quedan le queda el menor tiempo, y planifica ese. Por lo tanto, en nuestro ejemplo, STCF reemplazara A y ejecutara B y C hasta completarse; solo entonces podra planificarse el tiempo que le queda a A.</br>

![STCF simple example](../Imagenes/STCF-simple-example.png)

&emsp;Como resultado, hay un tiempo de entrega promedio mucho mejor: 50 segundos</br>

$$\frac{(120 - 0) + (20 - 10) + (30 - 10)}{3} = 50$$

&emsp;Y como antes, dadas nuestras suposiciones, STCF es comprobablemente optimo; dado que SJF es optimo si todos los procesos llegan al mismo tiempo, probablemente serias capaz de ver la intuicion detras de la optimizacion de STCF.</br>

### Una nueva metrica: Tiempo de respuesta (Response Time)

&emsp;Si conocemos la longitud de los procesos, y que procesos usarian solo la CPU, y nuestra unica metrica fuera el tiempo de entrega, STCF seria una gran politica. De hecho, para un numero de los primeros sistemas de computacion por lotes, ese tipo de algoritmos de planificacion tenian sentido. Sin embargo, la introduccion de maquinas de tiempo compartido cambio eso. Ahora los usuarios se podian sentar en una terminal y demandar un desempeño interactivo del sistema. Y por lo tanto, una nueva metrica nacio, **tiempo de respuesta (response time)**.</br>
&emsp;Definimos el tiempo de respuesta como el tiempo desde que el proceso llego al sistema hasta el tiempo en que fue planificado por primera vez. Mas formalmente:</br>

$$T_{response} = T_{firstrun} - T_{arrival}$$

&emsp;Por ejemplo, si tenemos la planificacion como en el caso anterior (Con A llegando al 0, y B y C al 10), el tiempo de respuesta de cada proceso seria: 0 para A, 0 para B, y 10 para C (promedio 3.33)</br>
&emsp;STCF y las disciplinas relacionas no son buenas para el tiempo de respuesta. Si, por ejemplo, los tres procesos llegan a la vez, el tercero tendria que esperar a que los primeros dos terminen antes de poder ejecutarse una vez. Mientras que es bueno para el tiempo de entrega, este enfoque es malo para el tiempo de respuesta y la interactividad. Como podriamos construir un planificador que sea sensible al tiempo de respuesta?</br>

### Round Robin

&emsp;Para resolver este problema, introduciremos un nuevo algoritmo de planificacion, clasicamente nos referimos a el como planificacion **Round-Robin (RR)**. La idea basica es simple: en vez de ejecutar procesos hasta que finalicen, RR ejecuta un proceso por una porcion de tiempo (a veces llamada **planificacion cuantica (scheduling quantum)**) y entonces cambia al siguiente proceso en la cola. Hace esto repetidamente hasta que todos los procesos se completen. Por este razon, RR a veces es llamado **time-slicing**. Notar que la longitud del periodo de tiempo, tiene que ser multiplo del periodo de interrupcion; por lo tanto si el timer se interrupte cada 10 milisegundos, el periodo de tiempo pordria ser 10, 20, o cualquier otro multiplo de 10ms.</br>
&emsp;Para entender RR con mas detalles, vamos un ejemplo. Asusmamos que tres procesos A, B, y C llegan al mismo tiempo al sistema, u que cada uno quiere ejecutarse 5s. Un plnificador SJF ejecutara cada proceso hasta que se complete antes de ejecutar otro, ver la siguiente imagen.</br>

![SJF Again (Bad Time Response)](../Imagenes/SJF-again.png)

&emsp;En contraste, RR con un **time-slice** de 1s que ciclara a traves de los procesos rapidamente, ver la siguiente imagen.</br>

![Round Robin (God For Response Time)](../Imagenes/RoundRobin.png)

&emsp;El **tiempo de respuesta promedio de RR** es:</br>

$$\frac{0 + 1 + 2}{3} = 1$$

&emsp;Y el **tiempo de respuesta promedio de SJF** es:</br>

$$\frac{0 + 5 + 10}{3} = 5$$

&emsp;Como se puede ver, la longitud de la porcion del tiempo es critica para RR. Mientras mas corta se la porcion de tiempo, mejor sera el desempeño de RR bajo la metrica de tiempo de respuesta. Sin embargo, hacer la porcion de tiempo muy corta es problematico: derrepente el costo de cambio de contexto dominaria todo el desempeño. Por lo tanto, decidir la longitud de la porcion de tiempo representa un compromiso para un diseñador de sistemas, hacerlo lo sufucientemente largo para **amortizar** el costo del cambio de contexto sin hacerlo tan largo que el sistema ya no sea sensible al tiempo de respuesta.</br>
&emsp;Notar que el cambio de contexto no surge solamente del las acciones de guardado y recuperado de registros del OS. Cuando los programas se ejecutan, acumulan una gran cantidad de datos de estado en la cache de la CPU, TLBs, *branch predictors*, y otros tipos de hardware en el chip. Cambiar a otro proceso causa que este estado sea descargado y un nuevo estado relevante del nuevo proceso sea traido, lo cual tiene un notable costo de desempeño.</br>
&emsp;RR, con una razonable porcion de tiempo, es un excelente plnificador si el tiempo de despuesta es nuestra unica metrica. Pero que pasa cuando nuestro viejo amigo, el tiempo de entrega. Veamos de nuevo el ejemplo de arriba. A, B, y C,cada uno con un tiempo de ejcucion de 5s, llegan al mismo tiempo, y el planificador es RR con una porcion de tiempo de 1s. Podemos ver en la imagen que A termina al 13, B al 14, y C al 15, para un promedio de 14. Muy mal promedio!.</br>
&emsp;Entonces, no es una sorpresa, que RR es por supuesto la peor politica de nuestra metrica es el tiempo de entrega. Intuitivamente, esto tiene sentido: lo que esta haciendo RR es alargando cada procesos tanto como puede, esto sucede porque ejecuta un proceso un corto periodo de tiempo y cambia al siguiente. Dado que el tiempo de entrega solo se preocupa de cuando finaliza un proceso, RR esta cerca de ser pesimo, en muchos casos incluso pero que el simple FIFO.</br>
&emsp;Mas generalmente, cualquier politica (como RR) que es **equitativa**, es decir, que eventualemente dividira la CPU entre los procesos activos en una corta escala de tiempo, desempeñara peor en metricas como el tiempo de entrega. Por supuesto, este es un sacrificio heredado: si estas dispuesto a no ser equitativo, puedes ejecutar el proceso mas corto hasta que se complete, el el costo de tiempo de respuesta se elevara; y si encambio valoras la equidad, el tiempo de respuesta bajara, pero se elevara el tiempo de entrega. Este tipo de sacrificios con comunes en los sistemas;</br>
&emsp;Hemos desarrolado dos tipos de planificadores. El primer tipo (SJF, STCF) optimizan el tiempo de entrega, pero tienen un mal tiempo de respuesta. El segundo tipo (RR) optimizan el tiempo de respuesta pero es malo para el tiempo de entrega. Y aun tenemos dos suposiciones que necesitan suavizarce: la suposicion 4 (los procesos no hacen I/O), y la suposicion 5 (conocemos el tiempo de ejecucion de cada proceso).</br>

### Incorporando I/O

&emsp;Primero vamos a suavizar la suposicion 4 -- obviamente todos los programas hacen I/O. Imginemos que un programa no puede tomar una entrada, esto producira la misma salida todo el tiempo. IMaginemos uno sin salida: es el proverbio del arbol que se cae en el bosque (si un arbol en el bosque se cae, pero nadie lo escucha, hace ruido?), si nadie puede ver lo que hace el programa; es como si no se hubiera ejecutado.</br>
&emsp;Un planificador, claramente, tiene una decision que tomar cuando un proceso inicia una peticion de I/O; porque el proceso que se esta ejecutando actualmente no puede usar la CPU dirante la I/O; se **bloquea** esperando que se complete la I/O. Si la I/O en enviada al disco duro, el proceso sera bloqueado por algunos milisengundos o mas, dependiendo la carga real I/O del disco. Por lo tanto, el planificador deberia planificar otro proceso en la CPU en ese tiempo.</br>
&emsp;El planificador tambien debera tomar un descision cuando una I/O se completa. Cuando esto ocurre, se genera una interrupcion, y el OS cambio es el estado *blocked* del proceso que hizo la peticion a *ready*. Por supuesto , todavia debera decidir ejecutar el proceso o no. Como deberia tratar el OS a cada proceso?</br>
&emsp;Para entender mejor este problema, vamos a asumir que tenenos dos procesos, A y B, los cuales necesitan 50ms de tiempo de CPU. Sin embargo, hay una diferencia obvia: A se ejecuta por 10ms y emite una pecition de I/O (aqui asumimos que cada I/O tarda 10ms), cuando B se ejecuta por 50ms y no hace ninguna I/O. El planificador ejecuta primero A y despues B.</br>

![Poor Use of Resources](../Imagenes/poor-use-resources.png)

&emsp;Ahora asumammos que estamos tratando de construir un planificador STCF. Como explicaria un planificador de este tipo el hecho de que A se divida en 5 subtrabajos de 10ms, cuando B solo demanda 50ms de CPU? Claramente, ejecutar un proceso y recien despues otro sin tener en cuenta las I/O no tiene sentido</bt>
&emsp;Un enfoque comun es trata cada subproceso de 10ms de A como un proceso independiente. Por lo tanto, cuando el sistema comienza, su opcion seria, ya sea ejecutar 10ms A o ejecutar 50ms B. Con STCF, la opcion es clara: elejir la mas corta, en este caso A. Entonces, cuando la primer subproceso de A se completa, solo queda B y empieza a ejecutarse. Entonces un nuevo subproceso de A en mandado y reemplaza a B y se ejecuta por 10ms. Hacer esto permite la **superposicion (overlap)**, con la CPU siendo usada por otro proceso mientras espera que el I/O de otro proceso se complete; por lo tanto el sistema es mejor utilizado (de forma mas eficiente).</br>

![Overlap](../Imagenes/overlap.png)

&emsp;Asi es como un planificador incorpora I/O. Tratando cada uso de CPU como un proceso diferente, el planificador se asegura que los procesos son "interactivos" ejecutandolos frecuentemente. Mientras estos procesos intereactivos estan haciendo I/O, otros procesos que usan la CPU se ejecutan, por lo tanto es una mejor utilizacion del procesador.</br>

### Sin oraculo

&emsp;Con un enfoque basico a I/O, llegamos a nuestra suposicion final: que el planificador conoce el tiempo de ejecucion de cada proceso. Y como dijimos antes, es la peor suposicion que podiamos ahcer. De hecho, en un OS proposito general, el OS conoce muy poco sobre el tamaño de los procesos. Por lo tanto, como podriamos contruir un plinificador como SJF/STCF sin un conocimiento previo? Aun mas, como podriamos incorporar algunas de las ideas que hemos visto con el planificador RR para que el tiempo de respuesta tambien se bastante bueno?</br>

[Anterior](./Ejecucion-directa.md) [Siguiente](./Planificador-multinivel.md)
