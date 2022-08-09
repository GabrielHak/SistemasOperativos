# Parte II &rarr; Concurrencia

Temas:

* [Introducción](./Introduccion.md)
* [API de hilos](#interludio-api-de-hilos): &larr; Usted esta aqui

  * [Creación de hilos](#creación-de-hilos)
  * [Finalización de los hilos](#finalización-de-los-hilos)
  * [Candados](#candados)
  * [Variables de condición](#variables-de-condición)
  * [Compilando y ejecutando](#compilando-y-ejecutando)

* [Candados](./Candados.md)
* [Estructuras de datos sincronizadas](..)
* [Variables de condicion](..)
* [Semáforos](..)
* [Problemas tipicos de concurrencia](..)

Bibliografia: [OSTEP Cap - 27 Interlude: Thread API](https://pages.cs.wisc.edu/~remzi/OSTEP/threads-api.pdf)

## Interludio: API de hilos

&emsp;Este capítulo cubre brevenmente las partes principales de la API de hilos. Cada parte será explicada en los siguientes capítulos, luego de mostrar como se usa la API.</br>

### Creación de hilos

&emsp;Lo primero que tenemos que hacer para crear un programa multi-hilo es crear nuevos hilos, y por lo tanto debe existir alguna interfáz de creación de hilos. En POSIX, es fácil:</br>

```c
include <pthread.h>
int
pthread_create(pthread_t      *thread,
    const pthread_attr_t      *attr,
      void                    *(*start_routine)(void*),
      void                    *arg);
```

&emsp;Esta declaración puede lucir compleja, pero realmente no está tan mal. Tiene cuatro argumentos: `thread, attr, start_routine, arg`. El primero, `thread`, es un puntero a una estructura de tipo `pthread_t`; usaremos esta estructura para interactuar con el hilo, y por eso necesitamos pasarsela a `pthread_create()` para inicializarla.</br>
&emsp;El segundo argumento, `attr`, es usado para especificar cualquier atributo que deba tener el hilo. Algunos ejemplos incluyen setear el tamaño del stack o quizas información sobre la prioridad de planificación del hilo. Un atributo es inicializado con una llamada separada a `pthread_attr_init()`. Sin embargo, en muchos casos, los que viene por defecto están bien; en este caso, simplemente le pasaremos el valor `NULL`.</br>
&emsp;El tercer argumento es el más complejo, pero solamente pide qué función debería empezar a ejecutar el hilo. En C, a esto lo llamamos **puntero a una función**, y esto nos dice que se espera lo siguiente: un nombre de una función (`start_routine`), al cual se le pasa un solo argumento de tipo `void *`, y la cual retorna un valor de tipo `void *`.</br>
&emsp;En cambio, si una rutina necesita un argutmento de tipo `int`, en vez de un puntero a void, la declaración luciría así:</br>

```c
int pthread_create(..., // first two args are the same
                  void *(*start_routine)(int),
                  int arg);
```

&emsp;Y si en vez de eso la rutina toma un puntero a void, pero retorna un entero, luciría así:</br>

```c
int pthread_create(..., // first two args are the same
                  int (*start_routine)(void *),
                  void *arg);
```

&emsp;Finalmente, el cuarto argumento, `arg`, es exactamente el argumento para ser pasado donde el hilo comienza la ejecucion. TE debes preguntar: porqué necesitamos esos punteros a void? Bueno, la respuesta es simple: teniendo un puntero a void como argumento de la función `star_routine` nos permite pasar cualquier tipo de argumento; y teniendolo como un valor de retorno le permite al hilo retornar cualquier tipo de resultado.</br>
&emsp;Veamos un ejemplo en la siguiente figura:</br>

```c
#include <stdio.h>
#include <pthread.h>

typedef struct {
  int a;
  int b;
} myarg_t;

void *mythread(void *arg) {
  myarg_t *args = (myarg_t *) arg;
  printf("%d %d\n", args->a, args->b);
  return NULL;
}

int main(int argc, char *argv[]) {
  pthread_t p;
  myarg_t args = { 10, 20 };

  int rc = pthread_create(&p, NULL, mythread, &args);
  ...
}
```

&emsp;Acá solo creamos un hilo al que se le pasan dos argumento, empaquetados en un solo tipo que definimos nosotros (`myarg_t`). El hilo, una vez creado, puede simplemente castear este argumento al tipo deseado ydesempaquetar los argumentos como deseamos.</br>

### Finalización de los hilos

&emsp;El ejemplo de arriba nos muestra como crear un hilo. Sin embargo, que pasa si queremos esperar a que termine un hilo? Necesitamos hacer algo especial para esperar por su finalización; en particular, debemos llamar a la rutina `pthread_join()`.</br>

```c
int pthread_join(pthread_t thread, void **value_ptr);
```

&emsp;Esta rutina toma dos argumentos. El primero es de tipo `pthread_t`, y es usado para especificar por cual hilo esperar. Esta variable es inicializada en la creación del hilo; si lo mantienes cerca lo puedes usar para esperar que termine.</br>
&emsp;El segundo argumento es un puntero al valor de retorno que deseas obtener. Dado que una puede retornar cualquier cosa, esta definida para retornar un puntero a void; dado que `pthread_join()` cambia el valor del argumento pasado, necesitas pasar un puntero y no el valor.</br>
&emsp;Veamos otro ejemplo:</br>

```c
typedef struct { int a; int b; } myarg_t;
typedef struct { int x; int y; } myret_t;

void *mythread(void *arg) {
  myret_t *rvals = Malloc(sizeof(myret_t));
  rvals->x = 1;
  rvals->y = 2;
  return (void *) rvals;
}

int main(int argc, char *argv[]) {
  pthread_t p;
  myret_t *rvals;
  myarg_t args = { 10, 20 };
  Pthread_create(&p, NULL, mythread, &args);
  Pthread_join(p, (void **) &rvals);
  printf("returned %d %d\n", rvals->x, rvals->y);
  free(rvals);
  return 0;
}
```

&emsp;En el código, se crea un solo hilo, y se le pasan un par de argumentos a través de la estructura `myargt_t`. Para retornar los valores, se usa el tipo `myret_t`. Una vez que el hilo termina de ejecutarse, el hilo principal, el cual estuvo esperando dentro de `pthread_join()`, retorna, y puede acceder a los valores retornados del hilo.</br>
&emsp;Hay que notar un par de cosas de este ejemplo. Primero, muchas veces no tenemos que hacer todo eso de empaquetar y desempaquetar los argumentos. Por ejemplo, si simplemente creamos un hilosin argumentos, podemos pasarle NULL como argumento cuando creamos el hilo. De forma similar, podemos psar NULL en `pthread_join()` si no queremos preocuparnos del valor de retorno.</br>
&emsp;Segundo, si queremos pasar un solo valor, no necesitamos empaquetarlo:</br>

```c
void *mythread(void *arg) {
  long long int value = (long long int) arg;
  printf("%lld\n", value);
  return (void *) (value + 1);
}

int main(int argc, char *argv[]) {
  pthread_t p;
  long long int rvalue;
  Pthread_create(&p, NULL, mythread, (void *) 100);
  Pthread_join(p, (void **) &rvalue);
  printf("returned %lld\n", rvalue);
  return 0;
}
```

&emsp;Tercero, hay que ser muy cuidadoso en como son retornados los valores del hilo. Específicamente, nunca retornar un puntero al que ferfiera a algo del stack del hilo. Aca hay un ejemplo de una pieza de código peligrosa:</br>

```c
void *mythread(void *arg) {
  myarg_t *args = (myarg_t *) arg;
  printf("%d %d\n", args->a, args->b);
  myret_t oops; // ALLOCATED ON STACK: BAD!
  oops.x = 1;
  oops.y = 2;
  return (void *) &oops;
}
```

&emsp;En este caso la variable `oops` está en el stack de `mythread`. Sin embargo, cuando la retorna, el valor se desasgina, y por lo tanto, devlver un puntero a una variable sin asignar traerá una seria de malos resultados.</br>
&emsp;Finalmente, usar `pthread_create()` para crear un hilo, y enseguida llamar a `pthread_join()`, es una forma rara de crear un hilo. De hecho, hay una forma más facil de hacer lo mismo; llamado **llamada a procedimiento**. Claramente, usualmente crearemos más de un hilo y esperaremos a que se complete.</br>
&emsp;también habrás notado que no todos los códigos que usn muli-hilo usa join. Por ejemplo, un servidor web multi-hilo debe crear muchos hilos, y usarlos en el hilo principal para aceptar peticioines y pasarselas a los hilos, indefinidamente. Tales programas de larga vida no necesitan esperar. Sin embargo, un programa paralelo que crea hilos para ejecutar una tarea en particular (en parelelo) probablemente use join para asegurarse que esa tarea se realizó antes de salir o de pasar al siguiente paso de cálculos.</br>

### Candados

&emsp;Más allás de la creación y espera de los hilos, probablemente, lo siguiente set de funciones más útiles provistas por la libreria de Threads de POSIX son aquellas para proveer exclusión mutua a una sección crítica a través de **candados**. El par de rutinas más básico para usar para este propósito  son las siguientes</br>

```c
int pthread_mutex_lock(pthread_mutex_t *mutex);
int pthread_mutex_unlock(pthread_mutex_t *mutex);
```

&emsp;Estas rutinas son fáciles de usar y entender. Cuando tienes una sección de código que es una sección crítica, y necesita ser protegida para asegurar su correcta ejecución, los candados son útiles. Probablemente imagines un código como este:</br>

```c
pthread_mutex_t lock;
pthread_mutex_lock(&lock);
x = x + 1; // or whatever your critical section is
pthread_mutex_unlock(&lock);
```

&emsp;La intención del código es la siguiente: si no hay otro hilo que mantenga el candado cuando se llama a `pthread_mutex_lock()`, el hilo adxquirirá el candado y entrará en la sección crítica. De hecho, si otro hilo tiene el candado, el hilo que esta intentando tener el candado no retornará de la llamada hasta que lo adquiera. Por supuesto, muchos hilos pueden quedar trabados dentro de la funcion de adquicisión en un momento dado; solo los procesos con el candado pueden llamar a `unlock`.</br>
&emsp;Desafotunadamente, el código está roto, de dos formas importantes. El primer problema es la **la falta de inicialización adecuada**. Todos los candados deben ser inicializados apropiadamente para asegurar que tienen los valores correctos para iniciar y por lo tranto hacer el trabajo que deseamos cuando el `lock` y `unlock` son llamados.</br>
&emsp;Con los hilos de POSIX, hay dos formas de incializar los candados. Una forma es usar `PTHREAD MUTEX INITIALIZER`:</br>

```c
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
```

&emsp;Hacer esto setea el candado en los valores por defecto y hace que el candado sea usable. La forma dinámica para hacerlo es llamar a `pthread mutex init()`:</br>

```c
int rc = pthread_mutex_init(&lock, NULL);
assert(rc == 0); // always check success!
```

&emsp;El primer argumento para esta rutina es la dirección del mismo candado, y el segundo es un conjunto de argumentos opcional. Cualquiera de las dos formas funcione, pero usualmente usamos la forma dinámica. Tambien se tiene que hacer el correspondiente llamado a `pthread mutex destroy()`.</br>
&emsp;El segundo problema con el código es que falla al verifcar errores de código cuando llama a `lock` y `unlock`. Si tu código no verifica errores, los fallos sucederán de manera silenciosa, lo que podría resultar en que muchos hilos entren a la sección crítica. Minimamente, usa *wrappers*, que afirman que la rutina tuvo éxito:</br>

```c
// Keeps code clean; only use if exit() OK upon failure
void Pthread_mutex_lock(pthread_mutex_t *mutex) {
  int rc = pthread_mutex_lock(mutex);
  assert(rc == 0);
}
```

&emsp;Los programas más sofisticados (non-toy programs), no pueden simplemente salir cuando algo sale mal, deben verificar fallos y hacer algo apropiado cuando la lladama no resulta exitosa.</br>
&emsp;Las rutinas `lock` y `unlock` no son las únicas con la librería pthreads para interactuar con los candados. Otras dos rutinas de interes:</br>

```c
int pthread_mutex_trylock(pthread_mutex_t *mutex);
int pthread_mutex_timedlock(pthread_mutex_t *mutex,
                            struct timespec *abs_timeout);
```

&emsp;Estas dos llamadas son usadas en la adquisición de candados. La versión `trylock` retorna un fallo si el candado ya lo tiene otro hilo; la versión `timedlock` de adquisición retorna despues de untiempo o despues de adquirir el candado, lo que suceda primero. Por lo tanto, timelock con un tiempo de cerose convierte en el caso trylock. Esas versiones deberían ser evitadas; sin embargo, hay un par de casos donde evitar quedarse estancado en una petición de candado puede ser útil.</br>

### Variables de condición

&emsp;El otro gran componente de cualquier librería de hilos, y ciertamente el caso con hilos de POSIX, es la presencia de una **variable de condición**. Las variables de condición son útiles cuando algun tipo de señalización debe tomar lugar entre los hilos, si un hilo está esperando que otra haga algo antes de continuar. Se usan dos rutinas principales para que los hilos interactuen:</br>

```c
int pthread_cond_wait(pthread_cond_t *cond, pthread_mutex_t *mutex);
int pthread_cond_signal(pthread_cond_t *cond);
```

&emsp;Para usar una condición de variable, uno tiene que tener además un candado que este asociado con esa condición. Cuando llamamos a cualquiera de esas rutinas el candado debe estar obtenido.</br>
&emsp;La primer rutina, `pthread_cond_wait()`, pone al hilo llamador a dormir, y por lo tanto espera que otro hilo le mande una señal, usualmente cuando algo en el programa es cambiado que el hilo ahora dormido debe considerar. Un uso típico es algo así:</br>

```c
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
Pthread_mutex_lock(&lock);
while (ready == 0)
  Pthread_cond_wait(&cond, &lock);
Pthread_mutex_unlock(&lock);
```

&emsp;En este código, después de incializar el candado relevante y la condición, un hilo verifica si la variable `ready` ya ha sido seteada a algo más que cero. Si no, el hilo simplemente llama a la rutina wait para dormir hasta que otro hilo lo despierte.</br>
&emsp;El código que despierta al hilo, el cual se debe ejecutar en algun otro hilo, debe ser algo asi:</br>

```c
Pthread_mutex_lock(&lock);
ready = 1;
Pthread_cond_signal(&cond);
Pthread_mutex_unlock(&lock);
```

&emsp;Un par de cosas para tener en cuenta en esta secuencia de código. Primero, cuando señalizamos (asi como al modificar la variable `ready`), siempre nos tenemos que asegurar de tener el candado. Asi nos aseguramos de no introducir una condición de carrera en nuestro código.</br>
&emsp;Segundo, debes haber notado que la llamada a wait toma un candado como segundo parámetro, cuando la llamada a `signal` solo toma una condición. La razón para esta diferencia es que la llamada wait, asemás de poner al hilo llamador a dormir, libera el candado cuando pone a dicho llamador a dormir. Imagina si no lo hiciera: como haría el otro hilo para adquirir el candado y enviarle la señal para que se despierte? Sin embargo,  antes de retornar luego de ser despertado el `pthread_cond_wait()` readquiere el candado. Por lo tanto, asegurando así que cada vez que el hilo que está esperando se ejecuta entre la adquisición del candado al principio de la secuencia wait, y la libreación del candado al final, mantiende el candado.</br>
&emsp;Una última singularidad: el hilo que está esperando verifica la condición en un bucle while, en vez de en un simple if. Discutiremos este tema en detalle más adelante, pero en general, usar un while es lo más facil y seguro de hacer. Apesar de que reverifica la condición (quizás agregando costos), hay algunas implementaciones de pthread que despiertan al hilo; en tal caso, sino reverificaramos, el el hilo que está esperando continuará pensado que la condición ha sido cambiada cuando en realidad no fue así. Por lo tanto es seguro ver el hecho de despertar como un pista de que algo ha cambiado y no como un hecho absoluto.</br>
&emsp;Notar que a veces es tentador usar una variable para enviar señales entre hilos, en vez de una variable de condición asociada a un candado. Por ejemplo, podríamos reescribir el código de arriba para que el código de espera se vea algo así:</br>

```c
while (ready == 0)
  ; // spin
```

&emsp;El código asociado a la señalización podría ser algo así:</br>

```c
ready = 1;
```

&emsp;Nunca hagas esto por las siguientes razones. Primero, se desempeña mal en muchos casos. Segundo, es propenso a errores.</br>

### Compilando y ejecutando

&emsp;Todos estos códigos de ejemplo son relativamente fáciles de hacer y ejecutar. Para compilarlos, solamente debemos incluir el encabezado `pthread.h` en tu código. En la linea de linkeo debes linkear explícitamente la librería pthreads, agregando la bandera `-pthread`.</br>
&emsp;Por ejemplo, para compilar un simple programa multihilo, todo lo que debemos hacer es lo siguiente:</br>

```shell
prompt> gcc -o main main.c -Wall -pthread
```

&emsp;Siempre y cuando hayas incluido el encabezado al main.c, ahora deberías tener compilado exitosamente un programa concurrente.</br>

[Anterior](./Introduccion.md) [Siguiente](./Candados.md)
