# Parte I &rarr; Virtualizacion

Temas:

* [Procesos](./Procesos.md)
* [API de procesos](#interlude-api-de-procesos): &larr; Usted esta aqui

  * [System Call ```fork()```](#system-call-fork)
  * [System Call ```wait()```](#system-call-wait)
  * [System Call ```exec()```](#system-call-exec)
  * [Porque? Motivando la API](#porque-motivando-la-api)
  * [Usuarios y control de procesos](#usuarios-y-control-de-procesos)
  * [Herramientas utiles](#herramientas-utiles)

* [Ejecucion directa limitada](./Ejecucion-directa.md)
* [Planificacion](./Planificacion.md)
* [Planificacion multinivel](./Planificador-multinivel.md)
* [Espacio de direcciones](./Espacio-direcciones.md)
* [API de memoria](Virtualizacion-API-de-memoria.md)
* [El mecanismo de traduccion de direcciones](Virtualizacion-El-menismo-de-traduccion-de-direcciones.md)
* [Segmentacion](Virtualizacion-Segmentacion.md)
* [Administracion de espacio libre](Virtualizacion-Administracion-de-espacio-libre.md)
* [Paginacion](Virtualizacion-Paginacion.md)
* [TLBs](Virtualizacion-TBLs.md)
* [Archivo de intercambio, mecanismo y politica](Virtualizacion-Archivo-de-intercambio-mecanismos-politica.md)

Bibliografia: [OSTEP Cap 5 - Interlude: Processes API](https://pages.cs.wisc.edu/~remzi/OSTEP/cpu-api.pdf)

## Interlude: API de procesos

&emsp;En este interludio, vamos a discutir el proceso de creacion en los sistemas UNIX. UNIX presenta una de las formas mas intrigantes de crear un nuevo proceso con un par de system calls: ***fork()*** y ***exec()***. UNa tercera rutina, ***wait()*** puede ser usada por un proceso para esperar que finalice un proceso que el a creado.</br>

### System Call ```fork()```

&emsp;La system call ```fork()``` es usada para crear un nuevo proceso, Sin embargo, hay que tener cuidado: esta es ciertamente la rutina mas extraña que jamas invocaras.</br>

```txt
p1.c
```

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(int argc, char *argv[]) {
    printf("hello world (pid:%d)\n", (int) getpid());
    int rc = fork();
    if (rc < 0) {
        // fork failed
        fprintf(stderr, "fork failed\n");
        exit(1);
    } else if (rc == 0) {
        // child (new process)
        printf("hello, I am child (pid:%d)\n", (int) getpid());
    } else {
        // parent goes down this path (main)
        printf("hello, I am parent of %d (pid:%d)\n",
        rc, (int) getpid());
    }
    return 0;
}
```

```console
prompt> ./p1
hello world (pid:29146)
hello, I am parent of 29147 (pid:29146)
hello, I am child (pid:29147)
prompt>
```

&emsp;Veamos que sucede en **p1.c** con mas detalle. Cuando empieza a ejecutarse el proceso imprime un mensaje "hello world"; e incluido en ese mensaje hay un identificador de proceso (**process identifier**), tambien conocido como **PID**. El proceso tiene un PID de 29146; en los sistemas UNIX, el PID es usado para nombrar a los procesos si uno quiere hacer algo con el, como por ejemplo, detenerlo.</br>
&emsp;Ahora la parte interesante comienza. El proceso llama a la system call ```fork()```, la cual proporciona el OS como forma de crear un nuevo proceso. La parte rara: el procesos creado es (casi) *una copia exacta del proceso que lo creo*. Esto significa que para el OS ahora hay dos copias del programa p1 ejecutandose, y ambos estan por regresar de la system call ```fork()```. El proceso recien creado (llamado **hijo (child)**, en contraste del proceso creador **padre (parent)** no empieza a ejecutarse en el ```main()```, (notar que el mensaje "hello world" solo se imprimio una vez); mas bien, este cobra vida como si el mismo hubiera llamado a ```fork()```</br>
&emsp;Debes haber notado: el hijo no es una copia *exacta*. Especificamente, aunque ahora tiene su propia copia de espacio de direcciones (es decir, su propia memoria privada), sus propios registros, su propio PC, etc, el valor que retorna al creador de ```fork()``` es diferente. Especificamente, mientras el padre recive el PID del hijo recien creado, el hijo recive codigo de retorno de cero. Esta diferenciacion es util, porque hace simple escribir el codigo para manejar los dos casos diferente (codigo de arriba).</br>
&emsp;Tambien habras notado que la salida de p1.c no es deterministica. Cuando el proceso hijo es creado, ahora hay dos procesos activos en el sistema de los que preocuparnos: el padre y el hijo. Por simplicidad asumamos que estamos ejcutando el programa en un sistema con un solo CPU, entonces, el padre o el hijo podrian ejecutarse en ese punto. En el ejemplo de arriba se ejecuto el padre y por lo tanto imprimio el mensaje primero. En otros casos, podria haber ocurrido lo opuesto:</br>

```console
prompt> ./p1
hello world (pid:29146)
hello, I am child (pid:29147)
hello, I am parent of 29147 (pid:29146)
prompt>
```

&emsp;El organizador del CPU (de ahora en mas CPU scheduler o simplemente ```scheduler```) es el que determina que proceso se ejecutara en un tiempo determinado; como el scheduler es complejo, usualmente no podremos asumir que decision tomara, y que proceso se ejecutara primero. Este no determinismo nos conduce a algunos problemas interesantes, particularmente en programas multihilo.</br>

### System Call ```wait()```

&emsp;Hasta ahroa no hicimos mucho: solo creamos un hijo que imprime un mensaje y termina. A veces, resulta mas util que el padre espere a que el proceso hijo finalice lo que estaba haciendo. Esta tarea se logra con la system call ```wait()``` (o su hermana mas completa ```waitpid()```.)</br>

```txt
p2.c
```

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(int argc, char *argv[]) {
    printf("hello world (pid:%d)\n", (int) getpid());
    int rc = fork();
    if (rc < 0) {
        // fork failed
        fprintf(stderr, "fork failed\n");
        exit(1);
    } else if (rc == 0) {
        // child (new process)
        printf("hello, I am child (pid:%d)\n", (int) getpid());
    } else {
        // parent goes down this path (main)
        int rc_wait = wait(NULL);
        printf("hello, I am parent of %d (rc_wait:%d) (pid:%d)\n",
        rc, rc_wait, (int) getpid());
    }
    return 0;
}
```

```console
prompt> ./p2
hello world (pid:29266)
hello, I am child (pid:29267)
hello, I am parent of 29267 (rc_wait:29267) (pid:29266)
prompt>
```

&emsp;En este ejemplo, el proceso padre llama a ```wait()``` para retrasar su ejecutcion hasta que el hijo termine de ejecutarse. Cuando el hijo termina, ```wait()``` regresa al padre.</br>
&emsp;Agregando la llamada ```wait()``` en el codigo hace que la salida sea determinisca</br>
&emsp;Con este codigo, sabemos que el hijo siempre imprimira primero, ya que si primero se ejecuta el padre, antes de imprimir llama a ```wait()```; y la system call no retornara hasta que el hijo no finalice, y solo entonces continua su ejecucion e imprime por pantalla.</br>

### System Call ```exec()```

&emsp;La parte final e importante del proceso de creacion API es la system call ```exect()```. Esta system call es util cuando queremos poner en ejecucion un programa que (es diferente desde el programa de llamadas)? (that is different from the calling program). Por ejemplo, llamando a ```fork()``` en p2.c es solo util si queremos mantener en ejecucion copias del mismo programa. Sin embargo, a menudo vamos a querer ejecutar un programa diferente, y ```exec()``` hace justo eso.</br>

```txt
p3.c
```

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <sys/wait.h>

int main(int argc, char *argv[]) {
  printf("hello world (pid:%d)\n", (int) getpid());
  int rc = fork();
  if (rc < 0) {
    // fork failed; exit
    fprintf(stderr, "fork failed\n");
    exit(1);
  } else if (rc == 0) { // child (new process)
    printf("hello, I am child (pid:%d)\n", (int) getpid());
    char *myargs[3];
    myargs[0] = strdup("wc");
    // program: "wc" (word count)
    myargs[1] = strdup("p3.c"); // argument: file to count
    myargs[2] = NULL;
    // marks end of array
    execvp(myargs[0], myargs); // runs word count
    printf("this shouldn’t print out");
  } else {
    // parent goes down this path (main)
    int rc_wait = wait(NULL);
    printf("hello, I am parent of %d (rc_wait:%d) (pid:%d)\n",
    rc, rc_wait, (int) getpid());
  }
  return 0;
}
```

```console
prompt> ./p3
hello world (pid:29383)
hello, I am child (pid:29384)
  29  107 1030  p3.c
hello, I am parent of 29384 (rc_wait:29384) (pid:29383)
prompt>
```

&emsp;En este ejemplo, el proceso hijo llama a ```execvp()``` para ejecutar el programa ```wc```, el cual es un programa contador de palabras (word counter). De hecho, ejecuta ```wc``` en el archivo fuente **p3.c**, por lo tanto nos dice cuantas lineas, palabras y bytes hay en el archivo.</br>
&emsp;La system call ```fork()``` es rara; su compañera de crimen. ```exec()```, tampoco es muy normal. Lo que hace: dado el nombre de un ejecutable (ej. ```wc```), y algunos argumentos (ej. **p3.c**), carga el codigo y los datos estaticos de ese ejecutable y sobreescribe su segmento de codigo actual (y sus datos estaticos) con el nuevo programa; el heap y el stack y otras partes del espacio de memoria del programa son re-inicializadas. Entonces el OS simplemente ejecuta ese programa, pasandole todos los argunmentos como el ```argv``` del proceso. Por lo tanto, esta system call no crea un nuevo proceso; mas bien, tranforma el actual programa en ejecucion (antes **p3**) en un diferente programa en ejecucion (```wc```). Despues de la llamada a ```exec()```, es como si **p3** nunca se hubiera ejecutado; una llamada exitosa a ```exec()``` nunca retorna.</br>

### Porque? Motivando la API

&emsp;Porque creariamos una interfaz tan rara para lo que deberia ser un simple acto de creacion de un nuevo proceso? Bueno, resulta que, la separacion de ```fork()``` y ```exec()``` es escencial en la construccion de una shell UNIX, porque esto le permite al shell ejecutar codigo **despues** de la llamada ```fork()``` pero ***antes*** de la llamada ```exec()```; este codigo puede alterar el ambiente del programa que esta a punto de ejecutarse, y por lo tanto habilitar una variedad de caracteristicas interensantes para ser construido facilmente.</br>
&emsp;El shell es solo un programa de usuario. Te muestra un **prompt** y entonces espera a que tipees algo. Entonces tipeas un comando (es decir un programa ejecutable, mas algunos argumentos); en muchos casos, el shell decifra en que parte del **file system** se encuentra el ejecutable, llama a ```fork()``` para crear un nuevo proceso hijo para ejecutar el comando, llama a alguna variante de ```exc()``` para poner en ejecucion el comando, y enontces espera que el comando complete su ejecucion llamando a ```wait()```. When el hijo termina, el sheel regresa del ```wait()``` e imprime de nuevo el **prompt**, listo para el siguiente comando.</br>
&emsp;La separacion de ```fork()``` y ```exec()``` le permite al shell hacer un puñado entero cosas utiles mas facil. Por ejemplo:</br>

```console
prompt> wc p3.c > newfile.txt
```

&emsp;En el ejemplo de arriba, la salida del programa wc es redirigida en el archivo de salida *newfile.txt*. La forma en que el shell logra esto es bastante simple: cuando el proceso hijo es creado, antes de llamar a ```wait()```, el shell cierra la salida estandar y abre el archivo *newfile.txt*. Haciendo esto, cualquier salida del ```wc``` a punto de ser ejecutado sera enviada al archivo en vez de a la pantalla.</br>

```txt
p4.c
```

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <fcntl.h>
#include <sys/wait.h>

int main(int argc, char *argv[]) {
  int rc = fork();
  if (rc < 0) {
    // fork failed
    fprintf(stderr, "fork failed\n");
    exit(1);
  } else if (rc == 0) {
    // child: redirect standard output to a file
    close(STDOUT_FILENO);
    open("./p4.output", O_CREAT|O_WRONLY|O_TRUNC, S_IRWXU);

    // now exec "wc"...
    char *myargs[3];
    myargs[0] = strdup("wc");
    // program: wc (word count)
    myargs[1] = strdup("p4.c"); // arg: file to count
    myargs[2] = NULL;
    // mark end of array
    execvp(myargs[0], myargs); // runs word count
  } else {
    // parent goes down this path (main)
    int rc_wait = wait(NULL);
  }
  return 0;
}
```

```console
prompt> ./p4
prompt> cat p4.output
  32  109 846 p4.c
prompt>
```

&emsp;Este ejemplo hace exactamente eso. La razon por la que la redireccion funciona se debe a una suposicion sobre como el OS maneja los **file descriptors**. Especificamente, los sistemas UNIX empiezan buscando file descriptor libres en creo. En este caso, *STDOUT_FILENO* sera el primero en estar disponible y por lo tanto es asignado cuando ```open()``` es llmado. Subsecuentemente escrito por el proceso hijo en la salida estandar del file descriptor, por ejemplo para las rutinas como ```printf()```, se enritara transparentemente al rachivo recien abierto en vez de la pantalla.</br>
&emsp;Sguramente habras notado al menos dos cosas sobre esta salida. Primero, cuando se ejecuta p4, es como si nada hubiera pasado; el shell solo imprime el command prompt e inmediatamente esta listo para el siguiente comando. Sin embargo, este no es el caso, de hecho el programa p4 llamo a ```fork()``` para crear un nuevo hijo, y entonces ejecuto el programa ```wc``` a  travez de una llamada a ```execvp()```. No se ve impresa ninguna salida en la pantalla porque a sido redireccionada al archivo *p3.output*. Segundo, como veras, cunado ejecutamos ```cat``` con el archivo de salida, se imprime toda la salida esperada del ```wc```.</br>
&emsp;Los **pipes** de UNIX son implementados de una forma similar, pero con la system call ```pipe()```. En esta caso, la salida de uno de los procesos es conectada a un **in-kernel pipe**, es decir una cola, y la entrada de otro proceso es conectada al mismo pipe; por lo tanto, la salida de un proceso aparenta ser usada como la entrada del siguiente, y una larga y util cadena de comandos pueden ser insertadas juntas. Como un ejemplo simple, consideremos buscar una palabra en un archivo, y contar cuantas veces esta esa palabra; los los pipes y las utilidades ```grep``` y ```wc```, es facil; solo hay que tipear ```grep -o foo file | wc -l``` en un command prompt y maravillarse con el resultado.</br>

### Usuarios y control de procesos

&emsp;Mas alla de ```fork()```, ```exec()``` y ```wait()```, en las sistemas UNIX hay muchas otras formas de interactuar con los procesos. Por ejemplo, la system call ```kill()``` es usada para enviar señales a un proceso, incluyendo directivas como pausar, morir, y otros imperativos utiles. Por conveniencia, en muchas shell de UNIX, ciertas combinaciones de teclas son especificadas para llevar una señal especifica al actual proceso en ejecucion; por ejemplo, *control-c* envia una ***SIGINT*** (interrumpir) al proceso (normalmente terminandolo) y *control-z* envia una señal ***SIGTSTP*** (stop) pausando el proceso en el medio de la ejecucion.</br>
&emsp;El subsistema completo de señales proporciona una infraestructura rica para enviar enventos externos a los procesos, incluyendo formas para recivir y procesar esas señales en procesos individuales, y formas para enviar señales tanto a procesos individuales como a procesos grupales. Para usar esta forma de comunicacion, un procesos debe usar la system call ````signal()``` para "agarrar" varias señales; haciendolo se asegura que cuando un señal particulas es entregada a un proceso, este suspendera su ejecucion normal y y ejecutara una parte paricular de codigo en respuesta de la señal.</br>
&emsp;Naturalmente de esto surge una pregunta: quien puede enviar una señal a un proceso, y quien no?. Generalmente, los sistemas que usamos pueden tener multiples personas unsandolos al mismo tiempo; si una de esas personas puede arbitrariamente enviar siñales como *SIGINT* para interrumpir el proceso, la usabilidad y seguridad del sistema estaria comprometida. Como resultado, los sistemas modernos incluyen una fuerte concepcion de la nocion de **usuario**. El ususario, despues de ingresar un contraseña para establecer credenciales, inicia sesion para ganar accesos a los recursos del sistema. Entonces el ususario puede ejecutar uno o muchos procesos, y ejercer control total sobre ellos (pausarlos, matarlos, etc). Los usuarios generalmente solo pueden controlar sus propios procesos; el trabajo del OS es repartir recursos a cada usuario para cumplir con los objetivos generales del sistema.</br>

### Herramientas utiles

&emsp;Hay muchas herramientas de la linea de comandos que son utiles. Por ejemplo, usando el comando ```ps``` puedes ver que proesos se estan ejecutando; leer la **man page** para ver alguna banderas (flags) utiles para pasarle a ```ps```. La herramienta ```top``` tambien es bastante util, muestra por pantalla los procesos del sistema y cuanta CPU y otros recursos estan usando. Graciosamente, muchas veces cuando ejecutamos este comando, dice que el proceso que mas recursos consume; quizas eso sea algo egolatra. El comando ```kill``` puede ser usado para enviar arbitraiamente señales a procesos, y uno levemente mas amistoso ```killall```. Asegurense de usarlos cuidadosamente; si accidentalmente matas el *window manager*, la computadora que tenes en frente puede volverse un poco complicada de usar.</br>

[Anterior](./Procesos.md) [Siguiente](./Ejecucion-directa.md)
