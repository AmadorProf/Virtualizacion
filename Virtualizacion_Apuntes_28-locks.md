# 28 · Locks

> Apuntes propios sobre el capítulo 28 de OSTEP (*Locks*). Fuente: https://ostep.org

## 28.1 Qué pedimos a un lock

Un lock protege una sección crítica: `lock()` la abre (o bloquea al llamante si está cogido), `unlock()` la cierra. Tres criterios de evaluación:

1. **Corrección**: exclusión mutua real.
2. **Equidad**: ¿algún hilo puede esperar indefinidamente (inanición)?
3. **Rendimiento**: coste sin contención; coste con muchos hilos y muchas CPUs.

## 28.2 Intentos fallidos que enseñan

- **Deshabilitar interrupciones**: funciona en monoprocesador para el kernel, pero: requiere privilegio (un proceso hostil monopoliza), **no funciona con varias CPUs** (el otro núcleo sigue entrando), y pierde interrupciones. Solo se usa dentro del kernel para secciones cortísimas.
- **Un flag con load/store normales**: comprobar-y-poner el flag son dos operaciones; una interrupción entre ambas mete a dos hilos dentro. Sin ayuda atómica del hardware, no hay lock correcto (los algoritmos de solo-loads/stores tipo Peterson existen, pero son frágiles con los reordenamientos de memoria modernos y no se usan).

## 28.3 Las primitivas hardware

Instrucciones **atómicas** (todo-o-nada, incluso entre CPUs):

- **test-and-set** (xchg): escribe 1 y devuelve el valor anterior, atómicamente.
- **compare-and-swap (CAS)**: "si la dirección vale E, escribe N; devuelve lo que había". Más potente (base también del lock-free).
- **load-linked / store-conditional** (RISC: ARM, RISC-V): el store solo triunfa si nadie tocó la dirección desde el load.
- **fetch-and-add**: suma atómica que devuelve el valor previo.

## 28.4 Spinlock: girar hasta entrar

```c
void lock(lock_t *l)   { while (TestAndSet(&l->flag, 1) == 1) ; }  // girar
void unlock(lock_t *l) { l->flag = 0; }
```

- **Correcto** con la primitiva atómica.
- **Equidad**: ninguna garantía — un hilo puede girar para siempre.
- **Rendimiento**: en monoprocesador, desastroso (el que gira quema su rodaja entera mientras el dueño del lock ni siquiera corre); con varias CPUs y secciones cortas, razonable (el dueño está corriendo en otra CPU y suelta pronto).

**Ticket lock** (con fetch-and-add): cada hilo toma número (`ticket`) y espera a que el turno (`turn`) llegue a él. Garantiza **orden FIFO** → sin inanición. Es el patrón de la charcutería.

## 28.5 Dejar de girar: el papel del SO

Girar desperdicia CPU. Alternativas escalonadas:

- **yield()**: al no conseguir el lock, ceder la CPU. Mejor que girar la rodaja entera, pero con muchos hilos sigue habiendo tormenta de cambios de contexto, y no evita inanición.
- **Colas de dormidos (park/unpark)**: el lock mantiene una cola; quien no lo consigue **se duerme** (`park`), y el que libera **despierta al primero** (`unpark`). Detalles finos que trata el capítulo: la ventana entre "decidir dormirse" y "dormirse" (resuelta con `setpark` o pasándole el problema al kernel), y el "traspaso" directo del lock al despertado (que evita carreras pero puede regalar el lock a quien tardará en correr).
- **Linux: futex** (*fast userspace mutex*): el caso sin contención se resuelve **enteramente en espacio de usuario** con un CAS sobre un entero; solo si hay contención se llama al kernel (`futex_wait`/`futex_wake`) para dormir/despertar. Lo mejor de ambos mundos: mutex baratísimo si no hay pelea, sin espera activa si la hay. Así están hechos los mutex de pthreads/glibc.
- **Híbrido two-phase**: girar un poquito (por si el lock se libera ya) y dormir si no. Lo usan glibc con `PTHREAD_MUTEX_ADAPTIVE_NP` y muchos runtimes.

## 28.6 Evaluación final

| Lock | Sin contención | Con contención | Equidad |
|---|---|---|---|
| Spin (TAS) | Muy barato | Malo (giro), cache-storm | No |
| Ticket | Muy barato | Giro pero FIFO | Sí |
| yield | Barato | Muchos switches | No |
| Cola + park | Algo más | Bueno | Sí |
| Futex/two-phase | Muy barato | Bueno | ~Sí |

## Ampliación (no está en OSTEP)

- El spinlock TAS ingenuo además **machaca el bus**: N CPUs girando con escrituras atómicas invalidan la línea de caché sin parar. Paliativos: **test-and-test-and-set** (girar leyendo, intentar escribir solo al verlo libre), *backoff* exponencial, y locks de colas **MCS/CLH** (cada CPU gira sobre su propia línea de caché — estándar en kernels; el qspinlock de Linux es un MCS compacto).
- **Priority inversion** (mencionado en un aside del capítulo y célebre por el Mars Pathfinder, 1997): hilo de baja prioridad tiene el lock, uno de media lo desplaza, el de alta espera el lock → el de alta corre "detrás" del de media. Soluciones: **herencia de prioridad** (`PTHREAD_PRIO_INHERIT`, los rt-mutex de Linux) o techo de prioridad.
- Más allá de locks: **lock-free** con CAS (cap. siguiente lo roza), **RCU** (lectores sin lock alguno; omnipresente en el kernel Linux — Turing-worthy idea de McKenney), y locks de lectura/escritura (cap. 31).

## Referencias

- OSTEP cap. 28.
- Mellor-Crummey & Scott, *Algorithms for Scalable Synchronization on Shared-Memory Multiprocessors*, ACM TOCS 1991 (MCS; premio Dijkstra).
- Franke, Russell & Kirkwood, *Fuss, Futexes and Furwocks*, OLS 2002 (futex).
- `man 2 futex`; `Documentation/locking/` del kernel.
