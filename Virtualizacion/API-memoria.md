# Parte I &rarr; Virtualizacion

Temas:

* [Procesos](./Procesos.md)
* [API de procesos](./API-de-procesos.md)
* [Ejecucion directa limitada](#mecanismo-ejecucion-directa-limitada)
* [Planificacion](#introduccion-planificacion-scheduling)
* [Planificacion multinivel](#planificador-la-cola-de-retroalimentacion-multinivel)
* [La abstraccion del espacio de direcciones](#la-abstraccion-espacios-de-memoria)
* [API de memoria](#interlude-api-de-memoria) : &larr; Usted esta aqui

  * [Tipos de memoria](#tipos-de-memoria)
  * [La llamada *malloc()*](#la-llamada-malloc)
  * [La llamada *free()*](#la-llamada-free)
  * [Errores comunes](#errores-comunes):

    * [Olvidar asignar memoria](#olvidar-asignar-memoria)
    * [No asignar suficiente memoria](#no-asignar-suficiente-memoria)
    * [Olvidar inicilizar la memoria asignada](#olvidar-inicializar-la-memoria-asignada)
    * [Liberar memoria antes de terminar de usarla](#liberar-memoria-antes-de-terminar-de-usarla)
    * [Liberar memoria repetidamente](#liberar-memoria-repetidamente)
    * [Llamar a free() de forma incorrecta](#llamar-a-free-de-forma-incorrecta)

  * [Soporte subyacente del OS](#soporte-subayciente-del-os)

* [El mecanismo de traduccion de direcciones](./Traduccion-direcciones.md)
* [Segmentacion](./Segmentacion.md)
* [Administracion de espacio libre](Virtualizacion-Administracion-de-espacio-libre.md)
* [Paginacion](Virtualizacion-Paginacion.md)
* [TLBs](Virtualizacion-TBLs.md)
* [Archivo de intercambio, mecanismo y politica](Virtualizacion-Archivo-de-intercambio-mecanismos-politica.md)

Bibliografia: [OSTEP Cap - 14 Cap 14 Interlude: Memory API](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-api.pdf)

## Interlude: API de memoria

&emsp;En este interlude, discutiremos las interfaces de asignacion de memoria en los sistemas UNIX. Las interfaces provistas son bastantes simples. El problema principal que trataremos es: Como asignar y manejar memoria?, Que infertaces son comunmente usadas? Que errores podriamos evitar?</br>

### Tipos de memoria

&emsp;En un programa en ejecucion en C, hay dos tipos de memorias asignadas. La primera es llamada **stack**, y las asignaciones y desasignaciones son manejadas *implicitamente* por el compilador para ti, el programador; por esta razon es llamada memoria *automatica*.</br>
&emsp;Declarar memoria en el stack en C es facil. Por ejemplo, digamos que necesitamos algo de espacio en la funcion *func()* para un entero llamado *x*. Para declarar una pieza de memoria, solo necesitas hacer algo como esto.</br>

```c
void func(){
  int x; //declares an integer on the stack
  ...
}
```

&emsp;El compilador hace el resto, asegurandose de hacer espacio en el stack cuando llames a *func()*. Cuando vuelves de la funcion, el compilador desasigna la memoria por ti; por lo tanto, si quieres que alguna informacion sobreviva mas alla de la invocacion, mejor no dejes esa informacion en el stack.</br>
&emsp;Es esta necesidad de memoria de larga vida que no lleva al segundo tipo de memoria, llamada **heap**, donde todas las asignacion y desasignaciones son manejadas por ti, el programador. Una gran responsabilidad, sin dudas! Y ciertamente la causa de muchos bugs. Pero si eres cuidadose y prestan atencion, usaras esas interfaces correctamente y sin demasiado problema. Aqui hay un ejemplo de como se debe asignar memoria para un entero en el *heap*.</br>

```c
void func(){
  int *x = (int *) malloc(sizeof(int));
}
```

&emsp;Un par de notas sobre este fragmento de codigo. Primero, habras notado que ambas asignaciones, stack y heap, ocurren en esta linea: primero el compilador sabe hacer espacio para un puntero a un entero cuando ve tu declaracion a dicho puntero *(int \*x)*; subsecuentemente, cuando el programa llama a *malloc()*, pide memoria para un entero en el heap; la rutina devuelve la direccion de dicho entero (ya sea exitoso, o NULL en caso de fallar), el cual es guardado en el stack para usar por el programador.</br>
&emsp;Dad su naturaleza explicita, y dado su uso mas variado, la memoria heap presenta mas retos para ambos, usuario y sistema. Por lo tanto, este es el foco de lo que queda de discucion.</br>

### La llamada *malloc()*

&emsp;La llamada **malloc()** es simple: consultas por espacio en el heap pasando el tamaño, y en cualquier caso te devuelve un puntero, al nuevo espacio asignado, o si falla devuelve *NULL*.</br>
&emsp;La pagina del manual te muestra que necesitas para usar malloc; tipea *man malloc* en la linea de comandos y veras esto</br>

```c
#include <stdlib.h>
...
void *malloc(size_t size);
```

&emsp;De esta informacion, puedes ver que todo lo que necesitas hacer es incluir el archivo encabezado *stdlib.h* para usar malloc. De hecho, realmente no necesitar hacerlo, como la libreria C, a la cual todo los programas estan enlazados por defecto, tiene adentro el codigo para *malloc()*; agregar el encabezado solo le permite al compilador verificar que estes llamando a *malloc()* correctamente (que estes pasendo el numero correcto de argumentos, del tipo correcto)</br>
&emsp;El unico parametro que toma *malloc()* es de tipo *size_t* el cual solamente describe cuantos bytes necesitas. Sin embargo, la mayoria de los programados no tipean directamente un numero; ya que es una pobre forma de hacerlo. En cambio, se usan muchas rutinas y macros para hacerlo. Por ejemplo, para asignar espacio para un valor flotante de doble precicion (*double*), siemplemente haz esto:</br>

```c
double *d = (double *) malloc(sizeof(double));
```

&emsp;Esta invoncacion a *malloc()* usa el operador *sizeof()* para solicitar la cantidad correcta de espacio; en C, esto suele pensarse como un operador *en tiempo de ejecucion*, lo que significa que el tamaño real se conoce *al momento de compilar* y por lo tanto un numero (en este caso 8, tamaño para un double) es sustituido como argumento en *malloc()*. Por eso razon, *sizeof()* es pensado como un operador y no como una llamada a una funcion (una llamada a una funcion toma lugar en tiempo de ejecucion.)</br>
&emsp;Tambien puedes pasarle el nombre de una variable (y no solo el tipo) a *sizeof()*, pero en algunos casos podrias no obtener los resultados esperados, entonces se cuidadoso. Por ejemplo, veamos el siguiente fragmento de codigo:</br>

```c
int *x = malloc(10 * sizeof(int));
printf("%d\n", sizeof(x));
```

&emsp;En la primera linea, declaramos espacio para un array de 10 integers, la cual esta elegante y fina. Sin embargo, cuando usamos *sizeof()* en la linea siguiente, devuelve un numero chico, algo como 4 (en maquinas de 32 bits) u 8 (en mauinas de 64 bits). La razon es que, en este caso, *sizeof()* cree que le estamos preguntando por el tamaño de un *puntero* a un entero, no sobre cuanta memoria tenemos asisgnada dinamicamente. Sin embargo, a veces, *sizeof()* funciona como se espera:</br>

```c
int x[10];
printf("%d\n", sizeof(x));
```

&emsp;En este caso, hay suficiente informacion estatica para que el compilador sepa que fueron asignados 40 bytes.</br>
&emsp;Otro caso con el que hay que tener cuidado, es con los strings. Cuando declaramos espacio para un string, usamos el siguiente formato: *malloc(strlen(s) + 1)*, el cual obtiene el largo del string usando la funcion *strlen()*, y le suma uno para hacer espacio para el caracter de fin se string. Usando *sizeof()* podria ser para problemas.</br>
&emsp;Tambien habras notado que *malloc()*  duelve un puntero a *void*.Hacer esto es la forma que tiene C para pasar un direccion dejarle al programador decidir que hacer con ella. El programador ademas se ayuda con lo que llamamos un **casteo (cast)**; en nuestro ejemplo arriba, el programa castea el tipo que devuelve *malloc()* a un puntero a *double*. Castear no es realmente complicado, ademas de decirle al compilador y a otros programadores "Si, se lo que estoy haciendo". Castear el resultado de *malloc()*, aunque no sea necesario, da seguridad.</br>

### La llamada *free()*

&emsp;Como resultado, asignar memoria es la parte facil de la ecuacion;saber cuando, como, e inclusi si liberar memoria es la parte dificil. Para liberar memoria del heap no es largo de hacer, los programadores simplemente llaman a ***free()***:</br>

```c
int *x = malloc(10 * sizeof(int));
...
free(x);
```

&emsp;La rutina toma como argumento un puntero retornado por *malloc()*. Como habras notado, el tamaño de la region asignada no es pasada por el usuario, y debe ser rastreada por la misma libreria de asignacion de memoria.</br>

### Errores comunes

&emsp;Hay ciertos errores comunes que surgen con el uso de *malloc()* y *free()*. Los siguientes ejemplos compilaron y el compilador no dijo ni pio.</br>
&emsp;El correcto majejo de la memoria ha sido un problema, de hecho, hay muchos lenguajes nuevos que tienen soporte de *manejo de memoria automatico*. En dichos lenguajes, llamas a algo parecido a *malloc()* para asignar memoria (usualmente **new** o algo similar para asignar un nuevo objeto), pero nunca tiene que llamar a algo como free para liberar espacio; mas bien, se corre un **recolector de basura (garbage collector)** y se da cuenta a que partes de la memoria no tienes referencias y la libera.</br>

#### Olvidar asignar memoria

&emsp;Muchas rutinas esperan memoria para ser asignada antes de que las lammes. Por ejemplo, la rutina *strcpy(dst, src)* copia un string de un puntero fuente a un puntero destino. Sin emabrgo, si no tienes cuidado podrias hacer algo como esto:</br>

```c
char *src = "Hello";
char *dst; //ops! unallocated
strcpy(dst, src); // segmentation fault and die
```

&emsp;Cuando ejecutas este codigo, probablemente conducira a un **segmentation fault (falla se segmentation)**, el cual es un termino lindo para **HICISTE ALGO MAL CON LA MEMORIA PROGRAMADOR IMPRUDENTE, ESTOY ENOJADO**</br>
&emsp;En este caso, el codig apropiado deberia ser algo como:</br>

```c
char *src = "Hello";
char *dst = (char *) malloc(strlen(src) + 1);
strcpy(dst, src); //work properly
```

&emsp;Como alternativa podrias usar *strdup()* y hacerte la vida incluso mas facil.</br>

#### No asignar suficiente memoria

&emsp;Un error relacionado es no asignar suficiente memoria, a veces llamado **buffer overflow**. En del ejemplo de arriba, un error comun es asignar *casi* el espacio suficiente para el buffer destino.</br>

```c
char *src = "Hello";
char *dst = (char *) malloc(strlen(src)); //to small!!
strcpy(dst, src); //work properly
```

&emsp;Por extraño que parezca, dependiendo de como este implementado malloc y muchos otros detalles, este programa podria ejecutarse aparentemente correctamente. En algunos casos, cuando se ejecuta la copia del string, escribe un byte mas lejos del final del espacio asignado, en algunos casos es inonfesivo, sobreescribiendo alguna variable que ya no se usa. En algunos casos, este overflow puede ser increiblemente dañino, y de hecho es la fuente de vulnerabilidades de seguridad en sistemas. En otros casos, la libreria de asignacion asigna un poco de espacio extra de todos modos, y por lo tanto tu programa realmente no sobreescribira otros valores de variables y funcionara bien. Incluso en otros casos, el programa en efecto fallara y se rompera.</br>

#### Olvidar inicializar la memoria asignada

&emsp;Con este error, puedes llamar a malloc de forma apropiada, pero olvidar de llenar tu tipo de dato recien asignado con un valor. Si te olvidas de hacerlo, eventualmente tu programa encontrara una **lectura sin asignar** cuando lea desde el heap algun dato de valor desconocido.</br>

#### Olvidar liberar memoria

&emsp;Otro error comun es conocido como **memory leak**, y esto ocurre cuando olvidas liberar la memoria. En los sistemas o aplicaciones de larga ejecucion es un gran problema, y no liberar memoria lentamente llevara a que te queden sin memoria, en el que sera neceseario reiniciar la maquina. POr eso, cuando terminas de usar un bloque de memoria asegurate de liberarlo. Notar que usar un lenguaje con garbage-collector no ayudara: si todavia tienen referencias a un bloque de memoria, ningun garbage-collector la liberara, y por lo tanto esa perdida de memoria traera problemas incluso en los lenguajes modernos.</br>
&emsp;En algunos casos, parece razonable no liberar memoria. Por ejemplo, si tu programa es corto, y terminara pronto; en ese casoi, cuando el programa finalice, el OS limpiara todas las paginas asignadas y no habra memory leaks en si. Mas alla de que esto funcione, es un mal habito al desarrollar. A la larga, uno de los objetivos como desarrollador es asquirir buenos habitos; uno de esos habitos es aprender como manejar memoria, y en lenguajes como C, liberar los bloques de meoria que asignamos.</br>

#### Liberar memoria antes de terminar de usarla

&emsp;A veces un programa leberara memoria antes de terminar de usarla; tal error se conoce como **puntero colgado (dangling pointer)**, y esto tambien es algo malo. Y su uso puede crashear el programa, o sobreescribir memoria valida (por ejemplo, llamaste a *free()* y despues a *malloc()* que reciclara la memoria recien liberada.</br>

#### Liberar memoria repetidamente

&emsp;Algun programa tambien liberan memoria mas de una vez; y es conocido como **double free**. El resultado de hacer esto esta indefinido. Y como pueden imaginar, la libreria de asignacion de memoria podria confundirse y hacer todo tipo de cosas raras; lo mas comun es crahear.</br>

#### LLamar a free() de forma incorrecta

&emsp;El ultimo problema que discutiremos es llamar a *free()* de forma incorrecta. Despues de todo, *free()* espera que le pases uno de los punteros que reciviste de un *malloc()*. Cuando le pasas algun otro valor, cosas malas pueden, y van a, pasar. Por lo tanto, ese **acceso invalido (invalid acces)** es peligroso y obviamente debe ser evitado.</br>

### Soporte subaycente del OS

&emsp;Como habras notada, hemos estado hablando de system calls cuando discutiamos *maloc()* y *free()*. La razon es simple: no son system calls pero si son library calls. Por lo tanto, la libreria de malloc maneja espacio en tu espacio de direcciones virtual, pero en si mismas estan construidas en lo alto de algunas system calls las cuales llaman al OS para pedir mas memoria o liberarla.</br>
&emsp;Dicha system call es llamada *brk*, la cual es usada para cambiar la locacion de quiebre del programa: la ubicacion del final del heap. Toma un argumento (la direccion del nuevo final), e incrementa o decrementa el tamaño del heap basado en si el nuevo final es mas grande o corto que el actual.</br>
&emsp;Notar que nunca vas a usar directamente *brk*. Estas son usadas por la libreria de asignacion de memoria; si las intentas usarlas, probablemente haras algo horriblemente mal. Apegate a *malloc()* y *free()*</br>
&emsp;Finalmente, tambien puedes obtener memoria del OS a traves de la llamada *mmap()*. Pasandole los parametros correctos, *mmap()* puede crear una region de memoria **anonima** en tu programa. Esta memoria puede ser tratada como un heap y manejada como tal.</br>

### Otras llamadas

&emsp;Hay mas llamadas que la libreria de asignacion de memoria soporta. Por ejemplo, *calloc()* asigna memoria y las incia en cero antes de retornarla; esto previene algunos errores cuando asumes que la memoria contiene ceros y te olvidas de inicializarla. La rutina *realloc()* puede ser util, , reserva una region de memoria nueva mas grande, copia el contenido de la region vieja, y retorna el puntero a la nueva region.</br>

[Anterior](./Espacio-direcciones.md) [Siguiente](./Traduccion-direcciones.md)
