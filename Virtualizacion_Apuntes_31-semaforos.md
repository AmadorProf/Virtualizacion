# 31 · Semáforos

> Apuntes propios sobre el capítulo 31 de OSTEP (*Semaphores*). Fuente: https://ostep.org

## 31.1 La primitiva de Dijkstra

Un **semáforo** es un entero con dos operaciones atómicas (nombres históricos P/V, en POSIX `sem_wait`/`sem_post`):

- `sem_wait(s)`: decrementa; si el valor queda negativo, **duerme**.
- `sem_post(s)`: incrementa; si hay dormidos, **despierta a uno**.

El valor inicial lo decide el programador y **codifica el comportamiento** — la clave de todo el capítulo. Intuición útil: un valor negativo = número de hilos esperando. A diferencia de las CVs, el semáforo **tiene memoria**: un post sin nadie esperando queda registrado (el contador sube) y el siguiente wait no duerme.

## 31.2 Los tres usos canónicos

**a) Lock binario** (valor inicial **1**): `wait` = lock, `post` = unlock. Con 2 hilos: el primero baja a 0 y entra; el segundo baja a −1 y duerme; el post lo libera.

**b) Ordenación / esperar un evento** (valor inicial **0**): el padre hace `wait`, el hijo al terminar hace `post` — un `join` casero. Funciona ejecute quien ejecute primero: si el hijo acaba antes, el post deja el valor en 1 y el wait del padre pasa de largo. (Con CVs esto requería lock + variable de estado + while; el semáforo lo empaqueta.)

**c) Regulador de concurrencia (throttling)** (valor inicial **N**): a lo sumo N hilos dentro de una región a la vez (p. ej. limitar cuántos hacen a la vez una fase que come mucha memoria).

## 31.3 Productor/consumidor con semáforos

Dos semáforos contadores + un mutex:

```c
sem_t vacios;  // = N  (huecos libres)
sem_t llenos;  // = 0  (items disponibles)
sem_t mutex;   // = 1  (protege el buffer)

// productor                    // consumidor
sem_wait(&vacios);              sem_wait(&llenos);
sem_wait(&mutex);               sem_wait(&mutex);
   poner(item);                    item = sacar();
sem_post(&mutex);               sem_post(&mutex);
sem_post(&llenos);              sem_post(&vacios);
```

El error didáctico del capítulo: poner el mutex **por fuera** de los semáforos contadores → un consumidor con el mutex cogido se duerme en `llenos`, y el productor que le daría el item se duerme en el mutex → **interbloqueo**. Regla resultante: el lock abraza *solo* la sección crítica, los semáforos de conteo van fuera.

## 31.4 Locks lectores/escritores

Muchos lectores pueden coexistir; un escritor exige exclusividad. Implementación clásica con contador de lectores: el **primer** lector coge el lock de escritura en nombre de todos, el **último** lo suelta. Pega estructural: los escritores pueden sufrir **inanición** si siempre hay lectores dentro. Las versiones justas existen pero complican; y el aviso del capítulo (citando a Hill: *big and dumb is better*): los rwlocks a menudo **no ganan** a un mutex simple por su propia sobrecarga — medir antes de sofisticar.

## 31.5 Los filósofos comensales

5 filósofos, 5 tenedores, cada uno necesita los dos adyacentes. Solución ingenua (todos cogen primero el izquierdo) → **interbloqueo circular**. Arreglo mínimo (Dijkstra): **romper la simetría** — el último filósofo coge primero el derecho → el ciclo de dependencias no puede cerrarse. Anticipo directo de las condiciones de deadlock (cap. 32).

## 31.6 Implementación y equivalencias

Un semáforo se construye con lock + CV + entero (ejercicio estándar, "Zemaphore" en el libro); y viceversa, CVs con semáforos es sorprendentemente sutil (buen ejercicio avanzado). Semáforos, locks y CVs son mutuamente expresables: elegir es cuestión de claridad, no de poder.

## Ampliación (no está en OSTEP)

- POSIX: `sem_init/sem_wait/sem_post` (con o sin nombre; los con nombre sirven **entre procesos**, `sem_open`). En Linux, sobre futex, cómo no.
- Uso moderno dominante del semáforo: **límites de recursos** (pools de conexiones, rate limiting, `Semaphore` de Java/Python asyncio); para exclusión mutua se prefiere el mutex (semántica más clara + herencia de prioridad, que el semáforo no puede tener: no sabe quién es su "dueño").
- La inanición de escritores es real en producción: los rwlocks de Linux kernel y `pthread_rwlock` tienen variantes con preferencia configurable (`PTHREAD_RWLOCK_PREFER_WRITER_NONRECURSIVE_NP`); la alternativa contemporánea para lectura masiva es **RCU** o *seqlocks*.
- Dijkstra introdujo los semáforos en el THE (1968) — el mismo sistema del que salió la estructura en capas; los papers EWD (dijkstra.utexas.edu) son legibles y curiosamente actuales.

## Referencias

- OSTEP cap. 31.
- Dijkstra, *Cooperating Sequential Processes* (EWD-123, 1968); *The Structure of the 'THE'-Multiprogramming System*, CACM 1968.
- Courtois, Heymans & Parnas, *Concurrent Control with Readers and Writers*, CACM 1971.
- `man 7 sem_overview`.
