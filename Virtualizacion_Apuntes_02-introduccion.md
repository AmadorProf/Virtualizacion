# 2 · Introducción a los sistemas operativos

> Apuntes propios sobre el capítulo 2 de OSTEP (*Introduction to Operating Systems*). Fuente: https://ostep.org

## 2.1 ¿Qué problema resuelve un SO?

Un programa en ejecución hace algo muy simple: busca la siguiente instrucción, la decodifica y la ejecuta (el **modelo de Von Neumann**). El sistema operativo existe para que ejecutar programas así sea **fácil, eficiente y seguro** cuando hay muchos programas, muchos usuarios y un solo hardware.

La herramienta universal del SO para lograrlo es la **virtualización**: tomar un recurso físico (CPU, RAM, disco) y transformarlo en una versión virtual más fácil de usar, más general y aparentemente privada. Por eso a veces se describe el SO como una **máquina virtual** en sentido amplio.

El SO ofrece sus servicios a través de **syscalls** (una **interfaz estándar**: unas ~300–400 llamadas en un Unix moderno), y por eso también se le describe como **gestor de recursos**: reparte CPU, memoria y disco entre quienes compiten por ellos.

## 2.2 Los tres pilares del libro

OSTEP organiza toda la asignatura en tres "piezas":

**1. Virtualización.** Dos preguntas concretas:

- ¿Cómo puede haber muchos programas "ejecutándose a la vez" con pocas CPUs? → virtualización de la CPU (capítulos 4–10). La demostración clásica: lanza el mismo bucle infinito 4 veces y los 4 progresan — el SO reparte la CPU en el tiempo.
- ¿Cómo puede cada programa tener "su" memoria empezando en la misma dirección? → virtualización de la memoria (capítulos 13–23). La demostración: dos procesos imprimen el mismo puntero (misma dirección virtual) y sin embargo escriben en sitios físicos distintos.

**2. Concurrencia.** Cuando varios hilos comparten memoria, las operaciones aparentemente atómicas (como `contador++`, que en realidad son tres instrucciones: load, add, store) se entrelazan y producen resultados incorrectos y no deterministas. Capítulos 26–33.

**3. Persistencia.** La RAM es volátil; los datos que importan deben sobrevivir a reinicios y fallos. El SO gestiona los dispositivos y ofrece el **sistema de ficheros**, con el problema estrella de qué pasa si se va la luz **a mitad de una actualización** (crash consistency). Capítulos 36–45.

## 2.3 Objetivos de diseño de un SO

- **Abstracciones buenas**: proceso, espacio de direcciones, fichero — hacen usable la máquina.
- **Rendimiento**: minimizar la sobrecarga que el propio SO añade (tiempo y memoria).
- **Protección/aislamiento**: el fallo (o la malicia) de un programa no debe tumbar a los demás ni al SO.
- **Fiabilidad**: el SO corre siempre; si él cae, cae todo.
- Otros según contexto: eficiencia energética, seguridad frente a atacantes, movilidad.

## 2.4 Un poco de historia

- **Al principio**: el "SO" era una biblioteca de rutinas comunes; se ejecutaba un trabajo cada vez (procesamiento por lotes), con operador humano.
- **Protección**: se descubre que el código de usuario no puede llamar a las rutinas del sistema como funciones normales → se inventa la **syscall** con **modo usuario/modo kernel** (Atlas, años 60): la llamada al sistema es una transferencia de control *controlada* que eleva el privilegio.
- **Multiprogramación** (era de los minicomputadores): varios trabajos en memoria a la vez; cuando uno espera E/S, corre otro → nacen los problemas de protección de memoria y concurrencia. **UNIX** (Thompson y Ritchie, Bell Labs, ~1969) sintetiza las buenas ideas de Multics en algo pequeño y elegante: todo es un fichero, programas pequeños componibles con pipes, escrito en C.
- **PC**: los primeros SO de PC (DOS, Mac OS clásico) *retrocedieron* — sin protección de memoria ni multitarea apropiativa; un programa podía tumbar el sistema. Con el tiempo, las ideas de los minis volvieron (Windows NT, Mac OS X — que es un Unix, Linux).

Moraleja histórica que el libro repite: **las buenas ideas del SO se inventaron pronto y se reinventan en cada plataforma nueva** (móviles, cloud…).

## Ampliación (no está en OSTEP)

- El número de syscalls de Linux hoy: ~460 en x86-64 (`man syscalls`). La tabla completa está en `arch/x86/entry/syscalls/syscall_64.tbl` del kernel.
- La discusión monolítico vs microkernel (Linux vs Minix, el famoso debate Torvalds–Tanenbaum de 1992) no aparece aquí pero es contexto útil: Linux es monolítico con módulos; los microkernels puros viven hoy en seL4, QNX (automoción) y como inspiración de los hipervisores tipo 1.

## Referencias

- OSTEP cap. 2 (y su bibliografía: el manual del Atlas de Kilburn et al., el paper original de UNIX de Ritchie & Thompson 1974).
- Ritchie & Thompson, *The UNIX Time-Sharing System*, CACM 1974 — legible y muy recomendable.
- `man 2 syscalls` en Linux.
