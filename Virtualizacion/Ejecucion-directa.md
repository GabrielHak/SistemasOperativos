# Parte I &rarr; Virtualizacion

Temas:

* [Procesos](./Procesos.md)
* [API de procesos](./API-de-procesos.md)
* [Ejecucion directa limitada](#mecanismo-ejecucion-directa-limitada): &larr; Usted esta aqui

  * [Tecnica basica: Ejecucion directa limitada](#tecnica-basica-ejecucion-directa-limitada)
  * [Problema #1: Operaciones restringidas](#problema-1-operaciones-restringidas)
  * [Problema #2: cambiando entre procesos](#problema-2-cambiando-entre-procesos):

    * [Un enfoque cooperativo: Esperar por las system calls](#un-enfoque-cooperativo-esperar-por-las-system-calls)
    * [Un enfoque no-cooperativo: El OS toma el control](#un-enfoque-no-cooperativo-el-os-toma-el-control)
    * [Guardando y restaurando el contexto](#guardando-y-restaurando-el-contexto)

  * [Preocupado sobre la concurrencia?](#preocupado-sobre-la-concurrencia)

* [Planificacion](./Planificacion.md)
* [Planificacion multinivel](./Planificador-multinivel.md)
* [La abstraccion del espacio de direcciones](./Espacio-direcciones.md)
* [API de memoria](./API-memoria.md)
* [El mecanismo de traduccion de direcciones](./Traduccion-direcciones.md)
* [Segmentacion](./Segmentacion.md)
* [Administracion de espacio libre](./Espacio-libre.md)
* [Paginacion](./Paginacion.md)
* [TLBs](./TLBs.md)
* [Archivo de intercambio, mecanismo y politica](Virtualizacion-Archivo-de-intercambio-mecanismos-politica.md)

Bibliografia: [OSTEP Cap 6 - Mechanism: Limited Direct Execution](https://pages.cs.wisc.edu/~remzi/OSTEP/cpu-mechanisms.pdf)

## Mecanismo: Ejecucion directa limitada

&emsp;Para virtualizar la CPU, el OS necesita de alguna forma compartir la CPU fisica entre los muchos trabajos ejecutandose aparentemente al mismo tiempo. La idea basica es simple: ejecutar un procesos por un tiempo pequeño, entonces ejecutar, y asi sucesivamente. Compartiendo el tiempo de CPU de esta manera, la virtualizacion se ha logrado.</br>
&emsp;Sin embargo, existen desafios en la construccion de tal maquinaria de virtualizacion. El primero es la *performace* (rendimiento): como podemos implementar la virtualizacion sin agregar costos excesivos al sistema? El segundo en el *control*: como podemos ejecutar procesos eficientemente mientras retenemos el control sobre la CPU? El control es paricularmente importante en el OS, si estuviera a cargo de los recursos; pero sin control, un proceso podria simplemente ejecutarse por siempre y tomar la maquina, o acceder a informacion a la que no deberia tener permitido el acceso. Obtener un alto desempeño mientras mantenemos control es uno de los retos centrales en la construccion de un OS.</br>

### Tecnica basica: Ejecucion directa limitada

&emsp;Para hacer que un programa se ejecute rapido como uno espera, no es de extrañar que los desarrolladores de OS idearan una tecnica, la cual llamamos **ejecucion directa limitada**. La parte de "ejecucion directa" de la idea es simple: solo ejecutar el programa ditectamente en la CPU. Por lo tanto, cuando el OS desea empezar a ejecutar un programa, crea una entrada de procesos en la lista de procesos para el, le asigna algo de memoria, carga el codgio del programa en la memoria (desde el disco), localiza su punto de entrada (es decir, la rutina ```main()``` or alguna similar), salta hasta ella, y comienza a ejecutar el codigo del usuario</br>

| Protocolo de ejecucion directa (sin limitaciones) |
| :---: |

| OS | Program |
| :---: | :---: |
| Create entry for process list | |
| Allocate memory for program | |
| Load program into memory | |
| Set up stack with argc/argv | |
| Clear registers | |
| Execute **call** main() | |
| | Run main() |
| | Execute **return** from main |
| Free memory of precess | |
| Remove from process list | |

&emsp;Suena simple, no? Pero este enfoque da lugar a varios problemas en nuestro camino a la virtualizacion de la CPU. El primero es simple: si solo corremos un programa, como el OS se asegura que el programa no hace nada que no queremos que haga, mientras aun lo ejecutamos de manera eficiente? El segundo: cuando estamos ejecutando un proceso, como hace el OS para pararlo su ejecucion y cambiar a otro proceso, por lo tanto, para implementar el tiempo compartido, necesitamos virtualizar la CPU?</br>
&emsp;Respondiendo las preguntas de arriba, tendremos una idea mejor de lo necesario para virtualizar la CPU. Desarrollando estas tecnicas, veremos de donde viene la parte de "limitada"; sin limites en un programa en ejecucion, el OS no estaria en control de nada y por lo tanto "solo seria una libreria" - un estado muy triste para un aspirante a sistema operativo.</br>

### Problema #1: Operaciones restringidas

&emsp;La ejecucion directa tiene la ventaja obvia de ser rapida; el program se ejecuta nativamente en el hardware del CPU y por lo tanto se ejecutara tan rapido como lo esperamos. Pero ejecutar un programa directo en la CPU introduce un problema: que pasa si el proceso quiere hacer algun tipo de operacion restringida, como emitir una solicitud de I/O a un disco, o ganar acceso a mas recursos del sistema como la CPU o la memoria</br>
&emsp;Un enfoque podria simplemente ser permitirle a cualquier proceso hacer cualquier cosa que ellos quieran en terminos de I/O y otras operaciones relacionadas. Sin embargo, hacer eso evitaria la construccion de muchos tipos de sistemas que son deseables. Por ejemplo, si queremos construir un sistema de archivos que verifique permisos antes de conceder acceso a un archivo, no podemos simplemente dejar a cualquier usuario procesar cualquier solicitud de I/O al disco; si lo hicieramos, un proceso podria simplemente leer o escribir el disco entero y por lo tanto todas las protecciones se perderian</br>
&emsp;Por lo tanto, el enfoque que tomamos es introducir un nuevo modo de procesador, conocido como **modo usuario**; el codigo que se ejecuta en el modo usuario es restringido en que puede hacer. Por ejemplo, cuando corremos en modo usuario, un proceso no puede emitir una peticios de I/O; hacerlo resultara en el surgimiento de una excepcion; el OS probablemente matara el proceso.</br>
&emsp;En contraste el modo usuario esta el **modo kernel**, en el cual se ejecuta el OS, el codigo que se ejecuta en ese modo puede hacer lo que quiera, inclutendo operaciones privilegiadas como eitir peticiones de I/O y ejecutar todos los tipos de instrucciones restringidas.</br>
&emsp;Sin embargo, todavia tenemos un reto: que deberia hacer un programa de usuario cuando quiere hacer algun tipo de operacion privilegiada, como leer desde el disco? Para poder hacer esto, virtualmente todos los hardwares modernos proporsionan a los programas de usuarios hacer una **system call**. Las system calls le permiten al kernel exponer cuidadosamente ciertas partes claves de funcionalidad a los programas de usuario, como acceder al sistema de archivos, crear y destruir procesos, comunicarse con otros procesos, y asignar mas memoria. Muchos OS proporsionan unas cientas system calls; al principio los sistemas UNIX exponian un subconjuto mas conciso de alrededor veinte system calls.</br>
&emsp;Para ejecutar una system call, un programa debe ejecutar una instruccion **trap** especial. Esta instruccion simultanemente salta al kernel y levanta el nivel de privilegios a modo kernel; una vez en el kernel, el sistema puede ejecutar cualquier operaciion privilegiada como necesite, y por lo tanto hacer el trabajo requerido para el proceso que lo pidio. Cuando termina, el OS llama a una funcion especial de **return-from-trap**, la cual, regresa al programa de usuario mientras simultanemente reduce el nivel de privilegios a modo usuario.</br>
&emsp;El hardware necesita ser un poco mas cuidadoso cunado ejecuta una instruccion **trap**, en el sentido que debe asegurarse de guardar susficientes registros del proceso que lo llama para poder regresar correctamente cuando el OS emita la instruccion **return-from-trap**. En x86 por ejemplo, el procesador empujara el PC, flags, y algunos otros registros en un **kernel stack** por proceso; el **return-from-trap** soltara esos valores del stack y continuara la ejecucion programa en modo usuario. Otros sistemas de hardware usan convenciones diferentes, pero el concepto basicos son similares en todas las plataformas.</br>
&emsp;Hay un detalle importante que dejamos fuera de la discucion: como sabe el trap que codigdo ejecutar dentro del OS? Claramente el proceso que lo llama no puede especificar a que parte de la memoria saltar (como lo hariamos en una llamada a un procedimiento); hacerlo le permitiria a los programas saltar a cualquier lugar dentro del kernel, lo que es una **Muy Mala Idea**. Por lo tanto el kernel debe controlar cuidadosamente que codigo ejecutar.</br>
&emsp;El kernel lo hace seteando una **trap table** cuando botea. Cuando la maquina bootea, lo hace en modo kernel, y por lo tanto el libre de configurar el hardware de la maquina como lo necesite. Una de las primeras cosas que el OS hace es decirle al hardware que codigo ejecutar cuando ocurre un evento expcional. Por, ejemplo. que codigo ejecutar cuando se produce una interrupcion de disco duro, que codigo ejecutar cuando ocurre una interrupcion de teclado, o cuando un programa hace una system call? El OS le informa al hardware la ubicacion de esos **trap handlers**, usualmente con algun tipo especial de instruccion. Una vez el hardware esta informado, recuerda la informacion de la ubicacion de esos manejadores hasta que la maquina es reiniciada, por lo tanto el hardware sabe que hacer (es decir, a que codigo saltar) cuando una system call y otros eventos toman lugar.</br>
&emsp;Para especificar la system call exacta, se le asigna a cada system cal un **numero de system call**. El codigo de usuario es el responsable de poner el numero de systam call deseada en un registro o en una ubicacion especifica en el stack; el OS, cuando maneja la system call dentro del trap handler, examina el numero, se asegura que sea valido, y si lo es, ejecuta el codigo correspondiente. Este nivel de indireccion sirve como forma de proteccion; el codigo de usuario no puede especificar una direccion de memoria a la que saltar, pero puede puede pedir un servicio particular a traves de un numero.</br>
&emsp;Ona ultima parte: ser capaces de ejecutar una instruccion para decirle al hardware donde estan las **trap tables** es una capacidad muy poderosa. Asique, como pudiste haber supuesto, tambien es una operacion privilegiada. Si intentas ejecutar esta instruccion en modo usuario el hardware no te dejera. Punto para reflexionar: que cosas horribles podrias hacerle a un sistema si pudieras instalar tu propia **trap table**? Podrias tomar la maquina?</br>

| OS @ boot (kernel mode) | Hardware |
| :---: | :---: |
| **initialize trap table** | |
| | remember address of ...|
| | syscall handler |

| OS @ run (kernel mode) | Hardware | Program (user mode) |
| :---: | :---: | :---: |
| Create entry for process list | | |
| Allocate memory for program | | |
| Load program into memory | | |
| Set up user stack with argv | | |
| Fill kernel stack with reg/PC | | |
| **return-from-trap** | | |
| | restore regs (from kernel stack) | |
| | move to user mode | |
| | jump to main | |
| | | Run main() |
| | | ... |
| | | Call system call |
| | | **trap** into OS |
| | save regs (to kernel stack) | |
| | move to kernel mode | |
| | jump to trap handler | |
| Handle trap | | |
| Do work of syscall | | |
| **return-from-trap** | | |
| | restore regs (from kernel stack) | |
| | move to user mode | |
| | jump to PC after trap | |
| | | ... |
| | | return from main |
| | | **trap** (via exit())|
| Free memory of process | | |
| Remove from process list | | |

&emsp;La linea de tiempo (de arriba hacia abajo) en la tabla anterior resume el protrocolo. Asumimos que cada proceso tiene un kernel stack donde los registros (incluyendo los registros de proposito general y el PC) son guardados para luego recuperarlos (por el hardware) entre las transisiones de entrada y salida del kernel</br>
&emsp;Hay dos fases en el protocolo de ejecucion directa limitada (**LDE**). La primera (al timpo de booteo), el kernel inicializa la trap table, y la CPU recuerda su ubicacion para su subsecuente uso. El kernel lo hace a traves de instrcciones privilegiadas (las que estan resaltadas en negrita).</br>
&emsp;La segunda (cuando se ejecuta un proceso), el kernel configura un par de cosas (asigna un nodo en la lista de procesos, asigna memmoria, etc) antes de usar la instruccion **return-from-trap** para ejecutar el proceso; esto cambia la CPU a modo usuario y comienza a ejecutar el proceso. Cuando el proceso quiere emitir una system call,  hace un **trap** de vuelta en el OS, el cual la maneja y una vez mas devuelve el control al proceso a traves de una funcion **return-from-trap**. Entonces el proceso completa su trabajo, y regresa del ```main()```; este retornara en alguna seccion de codigo que terminara el programa apropiadamente (es decir, llamando a la system call ```exit()```, la cual hara **trap** en el OS).</br>

### Problema #2: Cambiando entre procesos

&emsp;El siguiente problema con la ejecucion directa es lograr cambiar de proceso. Cambiar de proceso deberia ser simple, verdad? El OS deberia simplemente decir frenar un proceso y comenzar otro. Cual es el problema? Realmente es un poco mas complicado: especificamente, si un proceso se esta ejecutando en la CPU, por definicion el OS no se esta ejecutando. Y si no se esta ejecutando como lo haria? (ayuda: no puede). Claramente no hay forma de que el OS tome una accion si no se esta ejecutando en la CPU.</br>

#### Un enfoque cooperativo: Esperar por las system calls

&emsp;Un enfoque que muchos sistemas tomaron en el pasado es conocido como el enfoque cooperativo. En este estilo, el OS *confia* en que los procesos del sistema van a comportarse rasonablemente. Se asume que los procesos que van a ejecutarse por mucho tiempo, peridicamente le vana ceder el control al OS para que el OS pueda decidir si es tiempo de ejecutar alguna otra tarea.</br>
&emsp;POr lo tanto, debes prguntarte, como un amigable proceso transfiere el conttrol de la CPu en este mundo utopico? Resulta que muchos procesos tranfieren el control de la CPU al OS frecuentemente haciendo llamadas a system calls , por ejemplo, para abrir un archivo y subsecuentemente leerlo, o mander un mensaje a otra maquina, o crea un nuevo proceso. Muchos sistemas como este, a menudo incluyen una system call explicita, **yield**, la cual no hace nada excepto tranferir el control al OS para asi poder ejecutar otros procesos.</br>
&emsp;Las apliciones tambien le ceden el control al OS cuando ahcen algo ilegal. Por ejemplo, si una aplicacion divide por cero, o si trata de acceder a memoria que no es accesible. generara una **trap** al OS. Entonces, el OS tendra el control de la CPU de nuevo y probabolemente matara el proceso.</br>
&emsp;Por lo tanto, en un sitema de organizacion cooperativo, el OS retoma el control de la CPU esperando una system call o que una operacion ilegal de algun tipo tome lugar. Seguramente pensaras: este enfoque pasivo no es una mala idea? Que sucede si, por ejemplo, un proceso (sin querer o malintencinadamente) termina en un blucle infinito y nunca hace una system call? Que podria hacer entonces el OS?</br>

#### UN enfoque no-cooperativo: El OS toma el control

&emsp;Resulta que, sin la ayuda del hardware, el OS no puede hacer mucho cuando un proceso se rehusa a hacer una system call y devolver el control al OS. De hecho, en el enfoque cooperativo, el unico recurso cuando un proceso se traba en un bucle infinito es recurrir a la solucion mas antigua de todos los problemas en los sistemas computacionales: ***REINICIAR LA MAQUINA***. Por lo tanto, arrivamos a un nuevo problema de nuestra busqueda incial para tener control de la CPU. Como puede hacer el OS para tener el control de la CPU incluso si un proceso no esta siendo cooperativo?</br>
&emsp;La respuesta resulta ser simple y fue descubierta hace muchos años: un **interruptor de tiempo (time interrupt)**. Un timer puede ser programado para lanzar una interrupcion cada tantos milisegundos; cuando la interrupcion es lanzada, el actual proceso en ejecucion es detenido, y se ejecuta en el OS un manejador de interrupciones preconfigurado. En este punto el OS retoma el control, y por lo tanto puede hacer lo que le plazca: detener el actual proceso y ejecutar uno diferente.</br>
&emsp;Al igual que con las system calls, el OS debe informar al OS que codigo ejecutar cuando ocurre una interrupcion; por lo tanto, en el booteo, el OS hace exactamente eso. En segundo lugar, tambien durante el booteo el OS debe iniciar el contador, el cual, obviamente, es una operacion privilegiada. Una vez el temporazdor se ha iniciado, el OS puede estar seguro que eventualmente tendra el control, y por lo tanto el libre de ejecutar programas de usuario. El temporizador tambien se puede apagar (otra operacion privilegiada).</bra>
&emsp;Notar que el hardware tiene algunas reponsabilidades cuando ocurre una interrupcion, en particular, guardar suficiente informacion del estado del programa que estaba ejecutandose cuando se dio la interrupcion tal que una posterior instruccion **return-from-trap** sea capaz de reanudar el programa en ejecucion correctamente. Este conjunto de acciones es muy similar al comportamiento del hardware durante una system call explicita dentro del kernel, con varios registros que se guardan, y por lo tanto se restauran facilmente mediante la instruccion **return-from-trap**.</br>

#### Guardando y restaurando el contexto

&emsp;Ahora que el OS ha retomado el control, ya sea mediante system calls cooperativas, o a traves de interrupciones, tiene que tomar una decision : seguir ejecutando el proceso actual o cambiar a otro diferente. Esta decision es tomada por una parte del OS conocida como **scheduler**.</br>
&emsp;Si la decision fue cambiar de proceso, entonces el OS ejecuta una parte de codigo de bajo nivel a la que nos referiremos como **cambio de contecto (context switch)**. Un cambio de contexto es conceptualmente simple: todo lo que el OS tiene que hacer es guardar un par de registros del proceso que se esta ejecutando actualmente (en el stack del kernel) y recuperar algunos del proximo proceso a ejecutar (tambien el stack del kernel). Haciendo eso el OS se asegura que cuando se ejecute la instruccion **return-from-trap**, en vez de volver al proceso que estaba ejecutando, el systema retomara la ejecucion de otro proceso.</br>
&emsp;Para guardar el contexto del proceso actual que se esta ejecutando, el OS ejecutara codigo assembly de bajo nivel (hay assembly de alto nivel?) para guardar los registros de proposito general, PC, y el puntero el stack del kernel del proceso actual. AL cambiar de stack, el codigo ingresa la system call pra el codigo de cambio en el contexto de un proceso y retorna el contexto de otro. Entonces cuando el OS finalmente ejecuta la instruccion **return-from-trap**, el proximo proceso a ejecutar pasa a ser el actual proceso en ejecucion. Y por lo tanto se completa el cambio de contexto.</br>

| OS @ boot (kernel mode) | Hardware |
| :---: | :---: |
| **initialize trap table** | |
| | rembember address of... |
| | syscall handler |
| | timer handler |
| **start interrup timer** | |
| | start timer |
| | interruptt CPU in X ms |

| OS @ run (kernel mode) | Hardware | Program (user mode)|
| :---: | :---: | :---: |
| | | Procees A |
| | | ... |
| | **timer interrupt** | |
| | save regs(A) &rarr; k-stack(A) | |
| | move to kernel mode | |
| | jump to trap handler | |
| Handle trap | | |
| Call *switch()* routine | | |
| save regs(A) &rarr; proc_t(A) | | |
| restore regs(B) &larr; proc_t(B) | | |
| switch to k-stack(B) | | |
| **return-from-trap into(B)** | | |
| | restore regs(B) &larr; k-stack(B) | |
| | move to user mode | |
| | jumps to B's PC | |
| | | Process B |
| | | ... |

&emsp;En estos cuadros se ve la linea de tiempo completa. El proceso A esta ejecutandose y es interrupido por el **timer interrupt**. El hardware guarda sus registros y entra en el kernel. En el manejador de interrupciones, el OS decide cambiar la ejecucion del Proceso A al Proceso B. En este punto, llama a la rutina *switch()*, la cual guarda cuidadosamente los valores de los registros (del proceso A), y restaura los registros del Proceso B, y entonces cambia de contexto, especificamente cambiando el *stack pointer* para usar el stack del kernel del proceso B. Finalmente el OS returns-from-trap, el cual restaura los registros del Proceso B y comienza a ejecutarlo.</br>
&emsp;Notar que hay dos tipos de guardado/restaurado de registros que suceden durante este protocolo. El primero es cuando una interrupcion ocurre; en ese caso, los *registros de usuario* del programa en ejecucion son guardados implicitamente por el *hardware*, usando el stack del kernel de ese proceso. El segundo es cuando el OS decide cambiar de A a B; en este caso, los *registros del kernel* son explicitamente guardador por el *software* (es decir, el OS), pero esta vez en la estructura de proceso del proceso. La ultima accion hace que el sistema pase de ejecutarse como si estuviera **trapped** en el kernel de A, a como si estuviera **trapped**.</br>

### Preocupado sobre la concurrencia?

&emsp;Algunos de ustedes, como lectores atentos y reflexivos, quizas ahora estan pensando: "Mmm... que sucederia cuando, durante una system call, ocurre una interrupcion?" o "Que pasa cuando estas manjeando una interrupcion y ocurre otra? Eso no es dificil de manejar en el kernel?".</br>
&emsp;La respuesta es si, de hecho el OS debe preocuparte por lo que ocurre si, durante el manejo de una interrupcion o un trap, ocurre otra interrupcion. Esto de hecho, el el tema exacto de la **concurrencia**.</br>

[Anterior](./API-de-procesos.md) [Siguiente](./Planificacion.md)
