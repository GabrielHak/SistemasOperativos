# Introduccion

Temas:

* [Virtualizacion de la CPU](#virtualizacion-de-la-cpu)
* [Vistualiazcion de la memoria](#virtualizacion-de-la-memoria)
* [Concurrencia](#concurrencia)
* [Persistencia](#persistencia)
* [Objetivos de diseño](#objetivos-de-diseño)
* [Historia](#algo-de-historia)

&emsp;Un programa en ejecucion hace una simple cosa: ejecuta instruccions. Millones e incluso miles de millones por segundo, el procesador busca (**fetchs**) una instruccion de la memoria, la decodifica (decodes), y la ejecuta (**executes**). Despues de ejecutarla, el procesador se mueve a lsa siguiente instruccion y asi sucesivamente hasta que el programa finalmente se completa.
&emsp;Y como dice Wolovick:</br>

```c
while(true){
    fetch();
    decode();
    execute();
}
```

&emsp;El Sistema Operativo (**OS**) es un cuerpo de software que hace que ejecutar programas sea facil, y aparenta permitire ejecutar varios a la vez, le permite a los programas compartir memoria, interactuar con dispositivos, etc.</br>
&emsp;La principal forma en que el OS hace esto, es a traves de una tecnica llamada virtualizacion (**virtualization**). El OS toma recursos fisicos (como el procesador, la memoria, o un disco), y los transorma en una forma virtual de ellos mismos, mas general, poderosa y facil de usar. Por eso, a veces nos referimos al OS como maquina virtual (virtual machine).</br>
&emsp;Para permitirle a los usuarios poder decirle al OS que hacer, y por lo tanto hacer uso de esas herramientas de la maquina virtual (como ejecutar un programa, asignar memoria, o acceder a un archivo), el OS tambien provee algunas interfaces (APIs) que se pueden llamar (invocar). Un tipico OS, de hecho, exporta algunas cientas de llamadas al sistema (system calls) que estan disponibles para las aplicaciones. Dado que el OS provee esas llamadas para ejecutar programas, acceder a memoria y dispositvos, y otras acciones relacionadas, tambien a veces decimos que el OS provee una libreria estandar (standar library) para aplicaciones.</br>
&emsp;Finalmente, como la virtualizacion permite ejecutar muchos programas, compartiendo CPU, y muchos programas acceden concurrentemente a sus propias instrucciones y datos, compartiendo memoria, y varios programas acceden a dispositivos, compartiendo disco, y asi sucesivamente, el OS es conocido como un manejador de recursos (**resource manager**). Cada CPU, memoria y disco es un recurso; y el rol del OS es manejar esos recursos, haciendolo de manera eficiente, justa y por supuesto, con otras metas en mente.</br>

## Virtualizacion de la CPU

&emsp;Es una tecnica del OS que da la ilusion de que el sistema tiene un numero mas grande de CPUs. Convirtiendo una sola CPU, o pocas, en un numero aparentemente infinito de CPUs, y permitendo que parezca que muchos programas se ejecutan al mismo tiempo.</br>
&emsp;Si dos o mas programas se ejecutan a la vez en un tiempo determinado, *cual* de ellos se deberia ejecutar? De esto se ocupan las politicas (**policies**) del OS.</br>

## Virtualizacion de la Memoria

&emsp;La memoria es solo un array de bytes; para **leer** la memoria uno solo debe especificar una direccion para ser capaz de acceder a los datos guardados ahi; para **escribir** o **actualizar** memoria, solo debemos especificar los datos que queremos escribir en la direccion dada. </br>
&emsp;La memoria es accedida todo el tiempo mientras un programa se ejecuta. Un programa tiene todos sus estructuras de datos en memoria, y los accede a traves de varias instrucciones, como cargar (**load**) y guardar (**store**), y otras instrucciones especificas. Ademas los programas tienen sus instrucciones en la memoria, asique acceden a memoria en cada **fetch**</br>
&emsp;Cada programa tiene su propio espacio de memoria virtual (**virtual address space**) o simplemente **addres space**, el cual el OS de alguna forma mapea en la memoria fisica de la maquina. Una referencia de memoria de un programa en ejecucion no afecta al espacio de memoria de otros procesos o del mismo OS; en lo que respecta al programa en ejecucion el tiene toda memoria fisica. La realidad, sin embargo es que la memoria fisica es un recurso compartido, manejado por el OS</br>

## Concurrencia

&emsp;Otro tema principal en este libro es la concurrencia (**concurrency**). Usamos este termino conceptual para referirnos a una multitud de problemas, y deben ser abordados cuando trabajamos con varias cosas a la vez, es decir, concurrentemente. en el mismo programa. Los problemas de concurrencia surgen primero dentro del mismo OS.</br>
&emsp;Desafortunadamente, los problemas de concurrencia no estan limitados solo al OS. Realmente, los programas multi-hilo (**multi-threaded**) presentan los mismo problemas</br>

## Persistencia

&emsp;El tercer tema importante del curso es la persistencia (**persistence**). En el sistema de memoria, los datos se pueden perder facilmente, y dispositivos como DRAM guardan valores de manera volatil; cuando la energia se va o el sistema se rompe (crashea), cualquier dato en la memoria se pierde. Por lo tanto, necesitamos hardware y software para ser capaces de guardas datos de manera persistente (**persistently &rarr; persistence**). Dicho almacenamiento es fundamental para cualquier sistema ya que los usuarios se preocupan muchos sobre sus datos.</br>
&emsp;El hardware es algun tipo de dispositvo de entrada/salida (**input/output or I/O**); en los sistemas modernos, un disco duro (**hard drive**) es un repositorio commen para guardar informacion, aunque los discos de estado solido (**solid-state drives SSDs**) estan abriendose camino en la arena.</br>
&emsp;El software en el OS, que usualmente maneja el disco el llamado sistema de archivos (**file system**); por lo tanto es el resposable de almacenar cualquier archivo que el usuaria crea en los discos del sistema, de forma confiable y eficiente.</br>
&emsp;A diferencia de las abstracciones de CPU y memoria que provee el OS, no crea un disco virtualizado y privado para cada aplicacion, porque se asume que, muchas veces, los usuarios querran compartir informacion que hay en esos archivos.</br>

## Objetivos de diseño

&emsp;Ahora tenemos idea de que hace realmente el OS; toma recursos fisicos, como el CPU, memoria o disco, y los virtualiza. Maneja temas dificiles y complicados como la concurrencia. Y almacena archivos de forma persistence. Dado que queremos construir un sistema asi, tenemos que tener algunos objetivos en mente para ayudarnos a focalizar nuestro diseño e implementacion y hacer compensaciones segun sea necesario.</br>
&emsp;Unos de los objetivos mas basicos es construir abstracciones (**abstractions**) con el objetivos de hacer el sistema comodo y facil de usar. La abstraccion hace posible escribir un programa grande dividiendolo en piezas mas pequeñas y entendibles, escribir un programa asi en un lenguaje de alto nivel como C, sin pensar en assembly, escribir codigo en assembly sin pensar en las compuertas logicas, y construir un procesador a partir de compuertas sin pensar demasiado en los transistores.</br>
&emsp;Una meta en el diseño e implementacion de un OS es proveer un alto desempeño, otra forma de decirlo es minimizar los gastos del OS. Virtualizar y hacer el sistema facil de usar vale la pena, pero no cualquier costo; por lo tanto, debemos luchar por proveer virtualizacion y otras herramientas del OS sin gastos excesivos.</br>
&emsp;Estos gastos se presentan de algunas formas: tiempo extra (mas instrucciones) y espacio extra (en memoria o disco). Buscar soluciones que minimicen uno u otro problema, o ambos, es posible. La perfeccion, sin embargo, no siempre es alcanzable, algo que aprendemos a notar, y cuando se apropiado, tolerar.</br>
&emsp;Otro objetivo es proveer proteccion entre aplicaciones, y tambien entre el OS y las aplicaciones. Porque queremos que al ejecutar varios programas al mismo tiempo, asegurarnos que el mal comportamien, accidental o intencionado, de uno de esos programas no dañe a los demas; y ciertamente no queremos que una aplicacion afecte al OS (que afectaria a todos los programas que estan ejecutandose en el sistema). La proteccion es el corazon de unos de los principios fundamentales de un OS, el aislamiento (**isolation**); aislar procesos unos de otros es la clave de la proteccion y por lo tanto subyace en gran parte de lo que debe hacer un OS.</br>
&emsp;El OS atmbien debe estar ejecutandose sin parar, cuando falla, todas las aplicaciones que esten corriendo tambien fallan. Debido a esta dependencia. los OS se esfuerzan por proveer un alto grado de confiabilidad.</br>
&emsp;Otros objetivos que tienen sentido:

* Eficiencia de energia, es importante en nuestro creciente *green world*
* Seguridad, una extension de proteccion, sobre todo en la era del internet
* Movilidad, dado que los OSes corren en dispositovos cada vez mas pequeños

## Algo de historia

### Primeros Sistemas Operativos, solo librerias

&emsp;Al principio, los OSes no hacian mucho. Basicamente eran un conjunto de librerias de funciones comunmente usadas; por ejemplo, en vez de que cada programador tenga que escribir codigo de bajo nivel para manejar dispositvos I/O, el OS les proveia APIs, haciendoles la vida mas facil.</br>
&emsp;Usualmente, en estos viejos sistemas mainframe, se ejecutaba un programa a la vez, controlado por un operador humano. Mucho de lo que un OS moderno deberia hacer, era realizado por este operador</br>
&emsp;Ese modelo de computacion fue conocido como procesamiento por lotes (**batch processing**). Las computadoras no eran usadas de manera interactiva, a causa del costo: simplmente era muy caro permitirle a un usuario sentarse en frente de la computadora y usarla, ya muchas veces permaneceria inactivo lo que le costaria a la instalacion miles de millones por hora.</br>

### Mas alla de las librerias: Proteccion

&emsp;Al ir mas alla de ser una simple libreria de uso comun, los OSes tomaron un rol mas central en el manejo de las maquinas. Una de los mas importantes aspectos de esto fue darse cuenta que el codigo ejecutado en nombre del OS era especial; y tenia el control de los dispositivos, y por lo tanto debia tratarse se forma diferente al codigo de una aplicacion normal. De eso modo, implementar un sistema de archivos (para manejar tus archivos) como una libreria tiene poco sentido En cambio, se necesitaba algo mas.</br>
&emsp;Entocnes, la idea de system call fue inventada. En lugar de proporcionar rutinas del OS como una libreria (donde simplemente realiza una llamada de procedimiento (**procedure call**) para acceder a ella), la idea fue agregar un par de instruccioines de hardware especiales y un estado de hardware para hacer que la transicion al OS sea un proceso mas formal y controlado.</br>
&emsp;La principal diferencia entre una system call y una procedure call es que la system call tranfiera el control al OS y, al mismo tiempo, aumenta el nivel de privilegio del hardware. Las aplicaciones de usarios se ejecutan en lo que se refiere como *modo usuario* (*user mode*), lo que significa que el hardware restringe los que las aplicaciones pueden hacer; por ejemplo, una aplicacion ejecutandose en modo usuario no puede iniciar una peticion de I/O al disco, a ninguna pagina de memoria fisica, o enviar un paquete por la red. Cuando una system call es iniciada, usualmente a traves de una instruccion especial de hardware llamada **trap**, el hardware transfiere el control a un pre-especificado **trap handler** (que el OS configuro previamente) y simultaneamente eleva el nivel de privelegios a **kernel mode**.</br>
&emsp;En el modo kernel el OS tiene acceso total al hardware y puede hacer cosas como inciar peticiones de I/O o disponerle mas memoria fisica a un programa. Cuando el OS termina de realizar la peticion,  devuelve el control al usuario via una instruccion especial **return-from-trap**, el cual revierte el modo usuario, mientras que simultaneamente pasa el control de nuevo a donde habia quedado la aplicacion.</br>

### La era de la multiprogramacion

&emsp;Donde los OSes realmente despegaron fue en la era de la computacion mas alla del mainframe, la era de las minicomputadoras. Maquinas clasicas como la familia PDP hizo que las computadoras fueran mucho mas asequibles; por lo tanto en lugar de tener un main frame por oraganizacion grande, ahora un grupo mas pequeño de personas dentro de una organizacion probablemente podria tener su propia computadora, no es sorprendente que uno de los impactos de esa caida de costo fue un aumento en la actividad del desarrolador; gente mas inteligente pusieron sus manos en las computadoras, y por lo tanto, hicieron que los sistemas informaticos hicieran mas cosas interesantes y hermosas,</br>
&emsp;En particular, la multiprogramacion se convirtio en un lugar comun debido al deseo de hacer un mejor uso de los recursos de la maquina. En lugar de solo ejecutar un trabajo a la vez, el OS debia cargar un numero de trabajos en la memoria e intercambian entre ellos rapidamente, mejorando la utilizacion de la CPU. Este intercambio fuer particularmente importante por los dispositivos I/O eran lentos, tener a un programa esperando en la CPU mientras el I/O empezaba el servicio era una perdida de tiempo de CPU.</br>
&emsp;Temas como la proteccion de memoria se volvieron importantes; no querriamos que one programa sea capaz de acceder a la memoria de otro programa. Entender como tratar con los problemas de concurrencia introducidos por la multiprogramacion tambien fue critico; asegurarnos que el OS se comporte correctamente a pesar de la presencia de interrupciones es un gran reto.</br>

### La era moderna

&emsp;Mas alla de las mini computadoras llego un nuevo tipo de maquinas, mas baratas, mas rapidas, y para las masas: la computadora personal (**personal computer &rarr; PC**)</br>
&emsp;Desafortunadamente, para los OSes, la PC al principio represento un gran salto hacia atras, ya que los primeros sistemas olvidaron las lecciones aprendidas en la era de las minicomputadoras. Por ejemplo, los primeros OSes como DOS (Disk Operating System from Microsoft) no creia que la proteccion de memoria fuera importante; por lo tanto, una aplicacion maliciosa (o simplemente mal programada) podria garabatear toda la memoria. La primera generacion de Mac OS, v9 y anteriores, tomaron un enfoque colaborativo para la planificacion de trabajos; por lo tanto, un hilo que accidentalmente quedo trabado en un loop infinito podia romper todo el sistema, forzando un reinicio.</br>
&emsp;Afoprtunadamente,despues de unos años de sufrimiento, la caracteristicas viejas de los OSes de las mincomputadoras enontraron su camino en las computadoras de escritorio.</br>

[Anterior](Capitulos.md) [Siguiente](Virtualizacion.md)
