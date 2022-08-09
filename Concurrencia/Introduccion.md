# Parte II &rarr; Concurrencia

Temas:

* [Introducción](#concurrencia-introducción): &larr; Usted esta aqui

  * [¿Porqué usar hilos?](#¿porqué-usar-hilos)
  * [Un ejemplo: creación de un hilo](#un-ejemplo-creación-de-un-hilo)
  * [¿Porqué se vuelve peor?: Datos compartidos](#¿porqué-se-vuelve-peor-datos-compartidos)
  * [El corazón del problema: Planificador descontrolado](#el-corazón-del-problema-planificador-descontrolado)
  * [El deseo de atomicidad](#el-deseo-de-atomicidad)
  * [Un problema más: Esperando por el otro](#un-problema-más-esperando-por-el-otro)

* [API de hilos](./API-Hilos.md)
* [Candados](./Candados.md)
* [Estructuras de datos sincronizadas](..)
* [Variables de condicion](..)
* [Semáforos](..)
* [Problemas tipicos de concurrencia](..)

Bibliografia: [OSTEP Cap - 26 Concurrency: An Introduction](https://pages.cs.wisc.edu/~remzi/OSTEP/threads-intro.pdf)

## Concurrencia: Introducción

&emsp;Hasta ahora, hemos visto el desarrollo de las abstracciones básicas que hace el OS. Hemos visto como toma una sola CPU física y la convierte el múltiples CPUs virtuales, permitiendo, por lo tanto, la ilusión de que múltiples programa se ejecutan al mismo tiempo. Tambien hemos visto como crear la ilusión de una gran memorial virtual privada para cada proceso; esta abstracción del el espacio de direcciones le permite a cada programa comportarse como si tuviera su propia memoria cuando, por supuesto, secretamente el OS multiplexa direcciones de espacio a través de la memoria física.</br>
&emsp;En esta nota, introduciremos una nueva abstracción para un solo programa en ejecución: la de **hilo (thread)**. En vez de nuestra clásica visión de un solo punto de ejecución en un programa, un programa **multi-hilo** tiene más de un punto de ejecución. Quizás otra forma de pensar esto es que cada hilo es más como otro proceso separado, excepto por una diferencia: *comparten* el mismo espacio de direcciones y por lo tanto pueden acceder a los mismos datos.</br>
&emsp;El estado de un solo hilo es muy similar al de un proceso. Tiene un pC que rastrea de dónde está obtiendo instrucciones el programa. Cada hilo tiene su propio set de registros privado que utiliza para cálculos; por lo tanto, si hay dos hilos que se estan ejecutando en un solo procesador, cuando cambia de ejecutar T1 a ejecutar T2, se produce un cambio de contexto. El cambio de contexto entre hilos es muy similar al cambio de contexto entre procesos, como el estado de registros de T1 debe ser guardado y el estado de los registros de T2 deben ser restaurados antes de ejecutarlo.. Con los procesos, guardabamos el estado en un **bloque de control de procesos (PCB)**; ahora, necesitaremos uno o más **bloque de control de hilo (TCBs)** para guardas el estado de cada hilo de un proceso. Aca hay una diferencia mayor, aunque, en el cambio de contexto que hacemos entre hilos en comparación a los procesos: el espacio de direcciones permanece igual (es decir, no es necesario cambiar la page table que estamos usando).</br>
&emsp;Otra gran diferencia entre hilos y procesos concierne al stack. En nuestro modelo simple del espacio de direcciones de un proceso clásico (al que ahora podemos llamar proceso de un hilo), hay un solo stack, usumente residiendo abajo del espacio de direcciones.</br>
&emsp;Sin embargo, en un proceso multi-hilo, cada hilo se ejecuta independientemente y pueden llamar a varias rutinas para hacer lo que tienen que hacer. En vez de un solo stack en el espacio de direcciones, habrá uno por hilo. Digamos que tenemos un proceso multi-hilo que tiene dos hilos; el espacio de direcciones resultante de verá direfente:</br>

![Multi-Thread Address Space](../Imagenes/Multi-ThreadAS.png)

&emsp;En la figura podemos ver dos stacks esparcidos en el espacio de direcciones del proceso. Por lo tanto, cualquier asignación en el stack de variables, parámetros, valores de retorno, y otras cosas que pongamos en el stack será ubicado en lo que a veces llamamos **thread-local storage** (almacenamiento del hilo local), es decir, el stack del hilo relevante.</br>
&emsp;Tener dos stacks no es un problema dado que por lo general no son muy grandes (excepto en programas que usen mucha recursión).</br>

### ¿Porqué usar hilos?

&emsp;Resulta que hay dos grandes razones por las que deberías usar hilos. La primera es simple: **parelelismo**. Imagina que estan escribiendo un programa que hace cálculos en arrays grandes, por ejemplo, uniendo dos arrays grandes en uno solo, o incrementando el valor de cada elemento en el array por alguna cantidad. Si estas ejecutando en solo un procesador, la tarea es fácil: solo hay que hace cada operación y listo. Sin embargo, si estamos ejecutando el programa en un sistema de muchos procesadores, tenemos el potencial de acelerar el proceso considerablemente usando los procesador para que cada uno haga una porción del trabajo. La tarea de tranformar tu programa de un solo hilo en un programa que hace este tipo de trabajo en múltiples CPUs es llamado **paralelización**,  y usar un hilo por CPU para hacer este trabajo es una forma natural y típica para hacer que los programas sean más rápidos en el hardware moderno.</br>
&emsp;La segunda razón es un poco más sutil: evitar bloquear el progreso del programa dado la lentitud de I/O. Imagina que estas escribiendo un programa que hace diferentes tipos de I/O: ya sea esperar para enviar o recibir un mensaje, que se complete un I/O explícito al disco, o incluso que un page fault finalice. En vez de esperar, tu programa podría hacer algo más, incluyendo usar la CPU para cálculos, o lanzar las peticiones I/O. Usar hilos es una forma natural para evitar quedar estancado; mientras un hilo espera, el planificador de la CPU puede cambiar a otro hilo, el cual este listo para ejecutarse y hacer algo útil. Los hilos permiten superposición de I/O con otras actividades en un solo programa; como resultado, muchas aplicaciones modernas basadas en servidores usan hilos en sus implementaciones.</br>
&emsp;En cualquier caso, puedes usar múltiples procesos en lugar de hilos. Sin embargo, los hilos comparten el espacio de direcciones y es más facil compartir datos. Los procesos son mas una opción para tareas lógicas separadas donde no se necesitan compartir muchas estructuras de datos.</br>

### Un ejemplo: Creación de un hilo

&emsp;Digamos que queremos ejecutar un programa que crea dos hilos, y que cada uno haga un trabajo independiente, en este caso, imprimir "A" o "B":</br>

```c
#include <stdio.h>
#include <assert.h>
#include <pthread.h>
#include "common.h"
#include "common_threads.h"

void *mythread(void *arg) {
  printf("%s\n", (char *) arg);
  return NULL;
}

int
main(int argc, char *argv[]) {
  pthread_t p1, p2;
  int rc;
  printf("main: begin\n");
  Pthread_create(&p1, NULL, mythread, "A");
  Pthread_create(&p2, NULL, mythread, "B");
  // join waits for the threads to finish
  Pthread_join(p1, NULL);
  Pthread_join(p2, NULL);
  printf("main: end\n");
  return 0;
}
```

&emsp;La función main crea dos hilos, los cuales ejecutan la función ```mythread()```, aunque con diferentes argumentos. Una vez que un hilo es creado, puede ejecutarse de inmeditao, o ponerlo en la lista de "ready".</br>
&emsp;Después de crear los dos hilos, main llamaa ```pthread_join()```, que espera a que se complete un hilo en particular.</br>
&emsp;Examinemos el posible orden de ejecución de este programa de la siguiente imagen:</br>

![Thread Trace](../Imagenes/TradeTrace1.png)

&emsp;En el diagrama de ejecución el tiempo aumenta hacia abajo, dada una secuencia de instrucciones, dependiendo de que hilo decida ejecutar el planificador. Por ejemplo, una vez que se crea un hilo, se puede ejcutar directamente, lo cual dejaria el siguiente orden de ejcución:</br>

![Thread Trace](../Imagenes/TradeTrace2.png)

&emsp;También podríamos ver printeado B antes de A, is, digamos, que el planificador decide ejecutar T2 primero incluso si T1 se creo antes; no hay razón para asumir que el hilo que se crea antes se ejecutrará primero:</br>

![Thread Trace](../Imagenes/TradeTrace3.png)

&emsp;Una forma de ver la creació de hilos es más como hacer una llamada a una función, en vez de ejecutar la función y volver al llamador, el sistema crea un nuevo hilo de ejecutción para la rutina que esta siendo llamada, y se ejecuta independientemente del llamador, quizas antes de volver de la creació, o quizas mucho después.</br>
&emsp;Los hilos nos hacen la vida complicada: ya es dificil decir que se ejecutará y cuando! Las computador ya son suficientemente difíciles de entender sin concurrencia. Desafortunadamente, con la concurrencia, esto se vuelve peor, mucho peor.</br>

### ¿Porqué se vuelve peor?: Datos compartidos

&emsp;El ejemplo simple que mostramos arriab es util mostrando como se crean los hilos y como se puede ejecutar en diferente orden dependiendo del planificador. Lo que no muestra es como interactuan los hilos cuando comparten datos.</br>
&emsp;Imaginemos un ejemplo simple donde dos hilos quieren actualizar una variable global compartida:</br>

```c
#include <stdio.h>
#include <pthread.h>
#include "common.h"
#include "common_threads.h"

static volatile int counter = 0;

// mythread()
//
// Simply adds 1 to counter repeatedly, in a loop
// No, this is not how you would add 10,000,000 to
// a counter, but it shows the problem nicely.
//
void *mythread(void *arg) {
  printf("%s: begin\n", (char *) arg);
  int i;
  for (i = 0; i < 1e7; i++) {
    counter = counter + 1;
  }
  printf("%s: done\n", (char *) arg);
  return NULL;
}

// main()
//
// Just launches two threads (pthread_create)
// and then waits for them (pthread_join)
//
int main(int argc, char *argv[]) {
  pthread_t p1, p2;
  printf("main: begin (counter = %d)\n", counter);
  Pthread_create(&p1, NULL, mythread, "A");
  Pthread_create(&p2, NULL, mythread, "B");

  // join waits for the threads to finish
  Pthread_join(p1, NULL);
  Pthread_join(p2, NULL);
  printf("main: done with both (counter = %d)\n",
  counter);
  return 0;
}
```

&emsp;Hay varias notas sobre el código. Primero, agrupamos la creación de los hilos y las rutinas join para simplificar la salida en caso de un fallo; para un programa simple como este, queremos al menos notar cuando ocurre un error, pero no hacer nada demasiado inteligente (solo salir). Por lo tanto, ```Pthread_create()``` solo llama ```pthreade_create()``` y se asegura que el código de retorno sea creo, si no lo es, imprime un mensaje y sale.</br>
&emsp;Segundo, en vez de usar dos funciones separadas para los hilos, usamos una sola pieza de código, y le pasamos al hilo un argumento para que impriman letras diferentes.</br>
&emsp;Finalmente, y más importante, podemos ver que cada hilo esta tratando de sumar un número a la variable compartida `counter`,y haciendolo 10 millones de veces en un loop. Por lo tanto, el resultado final deseado es 20,000,000.</br>
&emsp;Ahora compilamos y ejecutamos el programa, para ve como se comporta. A veces, todo se comporta como esperamos:</br>

```shell
prompt> gcc -o main main.c -Wall -pthread; ./main
main: begin (counter = 0)
A: begin
B: begin
A: done
B: done
main: done with both (counter = 20000000)
```

&emsp;Desafortunadamente cuando ejecutamos este programa, incluso en u solo procesador, no necesariamente obtenemos el resultado deseado. A veces obtenemos:</br>

```shell
prompt> ./main
main: begin (counter = 0)
A: begin
B: begin
A: done
B: done
main: done with both (counter = 19345221)
```

&emsp;Intentemos una vez más, solo para ver si nos volvimos locos. Despues de todo, ¿no se supone que las computadores producen resultados determinísticos, como te enseñaron? ¿Quizás te estuvieron mintiendo?</br>

```shell
prompt> ./main
main: begin (counter = 0)
A: begin
B: begin
A: done
B: done
main: done with both (counter = 19221041)
```

&emsp;No solo cada programa se ejecuta mal, sino que también se produce un resultado diferente. ¿Porqué sucede esto?</br>

### El corazón del problema: Planificador descontrolado

&emsp;Para entender porqué sucede esto, debemos entender la secuencia de código que genera el compilador para actualizar la variable `counter`. En este caso, solo queremos sumar un número a `counter`. Por lo tanto, la secuencia de código para hacerlo luciría algo asi:</br>

```assembly
mov 0x8049a1c, %eax
add $0x1, %eax
mov %eax, 0x8049a1c
```

&emsp;Este ejemplo asume que la variable `counter` está en la dirección 0x8049A1C. En esta secuencia de tres instrucciones, la instruccion x86 `mov` se usa primero para obtener el valor de la memoria en la dirección y ponerla en un registro `eax`. Entonces, se hace la suma, y finalmente, el contenido de `eax` es guardado nuevamente en la misma dirección de la memoria.</br>
&emsp;Imaginemos que uno de nuestro hilos (T1) entra en esta regió de código, y esta por incrementar el contador en uno. Carga el valor del contador (digamos que esta en 50) en el registro `eax`. Por lo tanto, `eax = 50` para el hilo T1. Entonces le suma uno al registro; por lo tanto `eax = 51`. Ahora ocurre algo desafortunado: una interrupción; por lo tanto el OS guarda el estado del hilo en la TCB del hilo.</br>
&emsp;Ahora ocurre algo peor, se elije a T2 para ejecutar, y entra a la misma pieza de código. Ejecuta la primera instrucción, obtiendo el valor del contador y poniendolo en `eax`. El valor del contador todavia es 50, y por lo tanto `eax = 50`. Asumamos que ahora T2 ejecuta las siguientes dos instrucciones, incrementado `eax` en uno, y guardando el contenido de `eax` en el contador. Por lo tanto el valor de la variable global ahora es 52.</br>
&emsp;Finalmente, ocurre otro cambio de contexto, y T1 vuelve a ejecutarse. Y ejecuta la última instrucción, y guarda el valor en la memoria; el contador sigue en 51.</br>
&emsp;Lo que sucedió es que el código que incrementa el contado de ejecutó dos veces, pero al contador solo se le sumo un uno. Una correcta versión de este programa debería tener la variable contador igual a 52.</br>
&emsp;Veamos un rastreo detallado de la ejecución para entender mejor el problema. Asumamos para este ejemplo que el código de arriba esta cargado en la dirección 100 en la memoria, como en la siguiente secuencia (note for those of you used to nice, RISC-like instruction sets: x86 has variable-length instructions; this mov instruction takes up 5 bytes of memory, and the add only 3):</br>

```assembly
100 mov 0x8049a1c, %eax
105 add $0x1, %eax
108 mov %eax, 0x8049a1c
```

&emsp;Con estas suposiciones, sucede esto:</nr>

![Thread Problem](../Imagenes/ThreadProblem.png)

&emsp;Asumamos que el contador empieza con el valor 50, y lo rastriemos a través del ejemplo para asguranos que entendemos lo que está sucediendo. Lo que hemos demostrado acá se llama **condición de carrera**: el resultado depende de los tiempos de ejecución del código. Con algo de mala suerte, obtendremos un resultado incorrecto. De hecho, obtendremos resultados diferentes cada vez; por lo tanto, en vez de un buen resultado **determinista** (al que estamos acostumbrados de las computador), llamamos a este resultado **indeterminado**, donde no sabemos cómo será el resultado y por supuesto es probable que sea diferente a lo largo de las ejecuciones.</br>
&emsp;Dado que múltiples hilos ejecutando este código puede resultar en una condición de carrera, llamamos este código una **sección crítica**. Una sección crítica es una pieza de código que accede a una variable compartida y no debe ser ejecutado por más de un hilo.</br>
&emsp;Lo que queremos para este código es lo que llamamos **exclusión mutua**. Esta propiedad garantiza que si un hilo esta ejecutando dentro de la sección crítica, a los demás se les impedirá hacerlo.</br>

### El deseo de atomicidad

&emsp;Una forma de resolver este problema podría ser tener más instrucciones más poderosas que, en un solo paso, haga lo que necesitamos hacer y se remueva la posibilidad de una interrupción no deseada. Por ejemplo, que pasaría si tuviéramos una super isntrucción comoe esta:</br>

```assembly
memory-add 0x8049a1c, $0x1
```

&emsp;Asumamos esta instruccion que suma un valor a una ubicación de memoria, y que el hardware la ejecuta **atómicamente**; cuando la instrucción es ejecutada, realiza el camibio como queremos. Y no se interruptirá a la mitad de la instrucción, dado que es eso lo que nos garantiza el hardware: cuando ocurre una interrupción o no se ha ejcutado la isntrucción en absoluto o ya se terminó de ejecutar; no hay estado intermedio.</br>
&emsp;Atómicamente, en este contexto, significa "como una unidad", lo que tomamos como "todo o nada". Lo que queremos es ejecutar estas tres isntrucciones atómicamente:</br>

```assembly
mov 0x8049a1c, %eax
add $0x1, %eax
mov %eax, 0x8049a1c
```

&emsp;Como dijimos, si tuvieramos una sola instrucción que haga eso, la usaríamos. Pero en el caso general, no tenemos tal instrucción. Imagina que construimos un B-tree concurrente, y queremos actualizarlo; realmente queremos ayuda del hardware para una instrucción "actualizar B-tree atómicamente"? Probablementeno, al menos en un set de instrucciones sensato.</br>
&emsp;Por lo tanto, lo que queremos hacer es pedirle al hardware algunas instrucciones útiles sobre las cuales podamos construir lo que llamamos **primitivas de sincronización**. Usando la ayuda del hardware, en combinación con ayuda del OS, podemos construir código multi-hilo que acceda a secciones críticas de una forma sincronizada y controlada, y por lo tanto, seguramente produzca el resultado correcto.</br>

### Un problema más: Esperando por el otro

&emsp;Resulta que hay otra interacción que surge, donde un hilo debe esperar por otro para completar alguna acción antes de continuar. Estas interacciones surgen, por ejemplo, cuando un proceso ahce una I/O al disco y se pone a dormir; cuando se completa la I/O, el preceso necesita ser despertado para poder continuar.</br>
&emsp;Por lo tanto, en los siguientes capítulos, no solo estudiaremos como construir soporte para las primiticas de sincronización para soportar atomocidad, sino también mecanismo para soportar este tipo de interacciones sleeping/waking que es comú en los programas mulit-hilo.</br>

[Anterior](../Virtualizacion/Paginacion-Multinivel.md) [Siguiente](./API-Hilos.md)
