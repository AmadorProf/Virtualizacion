# 5 · Interludio: la API de procesos de Unix

> Apuntes propios sobre el capítulo 5 de OSTEP (*Interlude: Process API*). Fuente: https://ostep.org

## 5.1 El trío: fork(), exec(), wait()

Unix separa "crear un proceso" y "ejecutar un programa" en dos llamadas distintas. Esta decisión, rara a primera vista, es la que da su potencia al modelo.

### fork()

`fork()` crea un **clon casi exacto** del proceso llamante. Rarezas importantes:

- **Retorna dos veces**: en el padre devuelve el PID del hijo; en el hijo devuelve 0. Ambos continúan desde el mismo punto del código, y se distinguen mirando ese valor.
- El hijo recibe **copias** del espacio de direcciones, registros y descriptores (copias lógicas: hoy se implementan con copy-on-write, ver ampliación).
- La salida de padre e hijo es **no determinista**: el scheduler decide quién corre primero — primer contacto del libro con el no determinismo que explota en la parte de concurrencia.

```c
#include <stdio.h>
#include <unistd.h>
#include <sys/wait.h>

int main(void) {
    pid_t pid = fork();
    if (pid < 0) {            // error
        perror("fork"); return 1;
    } else if (pid == 0) {    // hijo
        printf("hijo: pid %d\n", getpid());
    } else {                  // padre
        wait(NULL);           // espera al hijo
        printf("padre: mi hijo era %d\n", pid);
    }
}
```

### wait() / waitpid()

El padre se bloquea hasta que un hijo termine, y recoge su código de salida. Además de sincronizar (hace determinista el orden de los printf de arriba), **libera al zombi**: hasta que alguien hace `wait`, el SO retiene la entrada del hijo terminado para poder entregar su estado.

### exec()

`exec()` (familia: `execl`, `execvp`, …) **reemplaza el programa del proceso actual** por otro: carga el nuevo ejecutable sobre el mismo proceso, reinicializa heap y pila, y salta a su `main`. No crea proceso nuevo y **no retorna si tiene éxito** — el programa que llamó ya no existe en ese proceso.

## 5.2 Por qué separar fork y exec: el shell

La ventana entre `fork()` y `exec()` permite al **hijo alterar su propio entorno antes de convertirse en el nuevo programa**, sin tocar al padre. Es exactamente lo que hace un shell con cada comando:

1. Lees `wc archivo.txt > salida.txt` y el shell hace `fork()`.
2. El **hijo**, antes de exec: cierra stdout y abre `salida.txt` (que reutiliza el descriptor 1) → la **redirección** queda montada.
3. El hijo hace `exec("wc", ...)`: wc escribe en "stdout" sin saber que es un fichero.
4. El padre hace `wait()` y muestra el prompt de nuevo.

Las **pipes** (`ls | wc -l`) son la misma jugada con `pipe()`: un buffer en el kernel con un extremo de escritura y otro de lectura; el shell conecta el stdout del primer hijo al extremo de escritura y el stdin del segundo al de lectura. Programas que no saben nada uno de otro quedan compuestos — la filosofía Unix hecha syscall.

```c
// esqueleto de un shell mínimo
while (1) {
    leer_linea(cmd);
    if (fork() == 0) {         // hijo
        preparar_redirecciones();
        execvp(cmd->argv[0], cmd->argv);
        perror("exec"); _exit(127);   // solo se llega aquí si exec falla
    }
    waitpid(-1, &status, 0);   // padre
}
```

## 5.3 Control de procesos y señales

- `kill()` envía **señales**: SIGTERM (terminar, educado), SIGKILL (matar, inapelable), SIGINT (Ctrl-C), SIGTSTP (Ctrl-Z: parar; se reanuda con `fg`/`bg`), SIGSEGV…
- Un proceso puede instalar manejadores con `signal()`/`sigaction()` para reaccionar (salvo a SIGKILL y SIGSTOP, incapturables).
- Solo puedes señalar procesos de tu mismo **usuario** (UID): la noción de usuario delimita el control.

## 5.4 Herramientas

`ps` (procesos), `top`/`htop` (en vivo), `kill`/`killall`, `pstree` (árbol padre-hijo), `nice`/`renice` (prioridad).

## Ampliación (no está en OSTEP)

- **fork() hoy es COW**: no se copia la memoria, solo las tablas de páginas, con todo marcado de solo lectura; la copia ocurre página a página al escribir (cap. 21 y `../virtualizacion.md` §3.3). Por eso `fork()` de un proceso de gigas cuesta microsegundos.
- **posix_spawn() y vfork()**: para el caso común fork+exec inmediato existen atajos; y hay una crítica académica seria a fork como API: Baumann et al., *A fork() in the road* (HotOS 2019) — argumenta que fork es un fósil hostil a la memoria masiva, los hilos y la seguridad. Contrapunto moderno al entusiasmo del capítulo.
- **Señales en tiempo real** (SIGRTMIN..SIGRTMAX) se encolan (las normales se fusionan). Y `signalfd`/`pidfd` son la interfaz moderna de Linux para gestionarlas sin las trampas clásicas de los handlers.
- El PID 1 (init/systemd) **adopta a los huérfanos** y les hace wait — por eso los zombis no se acumulan indefinidamente si el padre muere.

## Referencias

- OSTEP cap. 5.
- Stevens & Rago, *Advanced Programming in the UNIX Environment* (APUE) — la referencia exhaustiva de esta API.
- Baumann et al., *A fork() in the road*, HotOS 2019.
- `man 2 fork`, `man 2 execve`, `man 2 wait`, `man 7 signal`.
