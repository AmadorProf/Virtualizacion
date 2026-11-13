# 29 · Estructuras de datos concurrentes con locks

> Apuntes propios sobre el capítulo 29 de OSTEP (*Lock-based Concurrent Data Structures*). Fuente: https://ostep.org

## 29.1 El problema general

Con locks en la mano, ¿cómo se hace *thread-safe* una estructura de datos (contador, lista, cola, hash)? La respuesta trivial siempre funciona: **un lock grande alrededor de toda operación**. La pregunta interesante es la segunda: ¿**escala**? — es decir, ¿N hilos en N CPUs van más rápido que uno? Con un lock único, jamás: la estructura se serializa.

## 29.2 Contadores: del exacto al aproximado

- **Contador con un lock**: correcto; con 4 hilos machacándolo, *más lento* que con 1 (contención + ping-pong de la línea de caché).
- **Contador aproximado (sloppy counter)**: un contador **local por CPU** (cada uno con su lock) + uno **global**. Los hilos incrementan su local (sin pelear con nadie); periódicamente — al superar un umbral S — el local se vacía al global. Leer el global da un valor *aproximado* (retrasado hasta S×CPUs). S regula el compromiso: grande = escala perfecto, menos preciso; pequeño = lo contrario.

La lección generaliza: **particionar el estado por CPU/hilo y reconciliar de vez en cuando** es EL patrón de escalabilidad (aparece en asignadores de memoria por arena, estadísticas del kernel, contadores per-CPU de Linux…).

## 29.3 Listas enlazadas

- Versión un-lock: correcta y simple. Refinamiento del capítulo: minimizar lo que está bajo el lock (p. ej. el `malloc` del nodo nuevo puede ir fuera; solo la manipulación de punteros necesita exclusión) — también simplifica los caminos de error, fuente clásica de bugs.
- **Hand-over-hand (lock coupling)**: un lock **por nodo**; al recorrer, coges el del siguiente antes de soltar el del actual. Concurrencia teórica máxima… y en la práctica **más lento** que un lock (adquirir/soltar un lock por nodo cuesta más que lo que se gana). Moraleja explícita del capítulo: *más concurrencia de grano fino no implica más velocidad*; mide.

## 29.4 Colas y tablas hash

- **Cola de Michael & Scott (dos locks)**: un lock para la cabeza (dequeue) y otro para la cola (enqueue) + un **nodo tonto** (dummy) que separa ambas operaciones → productores y consumidores no compiten entre sí.
- **Tabla hash con un lock por bucket**: escala magníficamente — la contención se reparte entre B buckets — y es la demostración estrella del capítulo: la hash concurrente simple aplasta a la lista con un lock. Estructura y granularidad natural bien casadas.

## 29.5 Consejos finales del capítulo

Empezar simple (un lock); refinarlo **solo si hay un problema de rendimiento medido**. Evitar la optimización prematura (Knuth). Ejemplo con historia: Linux pasó del **Big Kernel Lock** a locks finos cuando las máquinas multi-CPU lo exigieron — no antes.

## Ampliación (no está en OSTEP)

- El siguiente escalón es prescindir de locks: estructuras **lock-free** con CAS (la cola M&S tiene versión lock-free canónica; ver Herlihy & Shavit, *The Art of Multiprocessor Programming*), y **RCU** para estructuras leídas masivamente y escritas rara vez (listas y hashes del kernel Linux: lectores sin ninguna barrera, escritores difieren la liberación).
- En la práctica de aplicación: usa estructuras concurrentes ya hechas — `ConcurrentHashMap` (Java, lock por segmento→CAS por bucket), `dashmap` (Rust), tcmalloc/jemalloc para el "contador sloppy" hecho asignador.
- **False sharing**: dos contadores per-CPU en la misma línea de caché se sabotean mutuamente aunque no compartan lock — alinear a 64 B (`alignas(64)`). El sloppy counter mal alineado no escala; el detalle de hardware manda (cap. 10 §coherencia).

## Referencias

- OSTEP cap. 29.
- Michael & Scott, *Simple, Fast, and Practical Non-Blocking and Blocking Concurrent Queue Algorithms*, PODC 1996.
- Herlihy & Shavit, *The Art of Multiprocessor Programming* — el libro del área.
- McKenney, *Is Parallel Programming Hard?* (libre online) — RCU y patrones per-CPU.
