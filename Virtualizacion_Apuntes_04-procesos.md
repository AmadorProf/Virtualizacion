# 4 · La abstracción: el proceso

> Apuntes propios sobre el capítulo 4 de OSTEP (*The Abstraction: The Process*). Fuente: https://ostep.org

## 4.1 Qué es un proceso

Un **proceso** es un programa en ejecución. El programa en disco es un objeto pasivo (bytes de código y datos, formato ELF en Linux); el proceso es ese programa **con vida**: con memoria asignada, registros cargados y el SO llevándole la cuenta.

El problema que resuelve la abstracción: dar la ilusión de **muchas CPUs** cuando hay pocas. La técnica es la **multiplexación en el tiempo** (*time sharing*): el SO ejecuta un proceso, lo detiene, ejecuta otro… Cada proceso cree tener la CPU para él; solo pierde algo de velocidad, no la ilusión.

Para construir la ilusión hacen falta dos cosas que el libro separa deliberadamente:

- **Mecanismos**: la maquinaria de bajo nivel — cómo se cambia de un proceso a otro (context switch, capítulo 6).
- **Políticas**: la inteligencia — *a cuál* le toca ahora (scheduling, capítulos 7–10).

Esta separación mecanismo/política es un principio de diseño general: permite cambiar la política sin tocar la maquinaria.

## 4.2 El estado de un proceso

Qué constituye un proceso — todo lo que puede leer o modificar mientras corre (su "estado de máquina"):

- **Memoria (espacio de direcciones)**: código, datos estáticos, heap (crece hacia arriba con `malloc`), pila (crece hacia abajo: variables locales, argumentos, direcciones de retorno).
- **Registros**: los de propósito general más tres especiales — el **contador de programa** (PC/RIP: qué instrucción toca), el **puntero de pila** (SP) y el frame pointer.
- **E/S**: los descriptores de fichero abiertos (en Unix, todo proceso nace con 0=stdin, 1=stdout, 2=stderr).

## 4.3 Creación de un proceso (qué hace el SO al "ejecutar un programa")

1. **Cargar** el código y los datos estáticos del ejecutable del disco a la memoria del proceso. Los SO modernos lo hacen **perezosamente**: mapean el fichero y traen páginas solo cuando se tocan (conecta con paginación, cap. 18–21).
2. Reservar y preparar la **pila** (con `argc`/`argv` colocados) y el **heap** inicial.
3. Tareas de inicialización de E/S: los tres descriptores estándar.
4. Saltar al punto de entrada (`main`, en realidad `_start` de la libc) → transferir la CPU al proceso.

## 4.4 Estados de un proceso

Modelo mínimo de tres estados:

```
            planificado (scheduled)
   LISTO ─────────────────────────→ EJECUTANDO
  (ready)←───────────────────────── (running)
            desplanificado                │ inicia E/S
              ↑                           ▼
              └──── E/S termina ──── BLOQUEADO
                                     (blocked)
```

- **Ejecutando**: tiene la CPU ahora.
- **Listo**: podría correr, pero la CPU está ocupada con otro.
- **Bloqueado**: no puede correr aunque hubiera CPU libre — espera un evento (típicamente E/S de disco o red).

La transición clave para el rendimiento: cuando un proceso se bloquea en E/S, el SO **aprovecha y ejecuta otro** (solapamiento de CPU y E/S). Decidir si retomar al que termina su E/S inmediatamente o no es decisión del scheduler.

Estados adicionales en sistemas reales: **inicial** (creándose), **zombi** (terminó, pero espera a que su padre recoja su código de salida con `wait()` — así el padre puede comprobar si acabó bien; el estado Z de `ps`).

## 4.5 Las estructuras de datos del SO

El SO es "solo un programa": rastrea los procesos con estructuras normales.

- **Lista de procesos** (process list / task list): una entrada por proceso.
- **PCB (Process Control Block)**: la estructura por proceso — estado, PID, registros guardados cuando no ejecuta (**contexto**), información de memoria, ficheros abiertos, padre, código de salida… En xv6 es `struct proc`; en Linux, `struct task_struct` (¡~700 campos!).

Cuando un proceso se desplanifica, sus registros se guardan en su PCB; restaurarlos después es lo que permite reanudarlo como si nada — el **cambio de contexto** (mecánica en el cap. 6).

## Ampliación (no está en OSTEP)

- En Linux la lista de procesos se explora en `/proc`: cada `/proc/<pid>/` es la vista pública del task_struct (`status`, `maps`, `fd/`…). Prueba: `cat /proc/self/status`.
- Linux no distingue estructuralmente proceso e hilo: ambos son `task_struct`; difieren en qué comparten (flags de `clone()`). Los "hilos del kernel" puros (kworker, ksoftirqd) aparecen entre corchetes en `ps`.
- Estados reales en Linux (`ps -o stat`): R (running/runnable), S (sleep interrumpible), D (sleep no interrumpible — la temida espera de disco que ni `kill -9` corta), T (parado), Z (zombi).

## Referencias

- OSTEP cap. 4 (y su bibliografía; destaca *The Nucleus of a Multiprogramming System*, Brinch Hansen 1970, origen del concepto de nucleus/kernel mínimo).
- Código de xv6 (`proc.h`, `proc.c`): un PCB legible en ~100 líneas. https://github.com/mit-pdos/xv6-riscv
- `man 5 proc`, `man 1 ps` en Linux.
