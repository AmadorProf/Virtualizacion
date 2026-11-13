# 10 · Planificación multiprocesador

> Apuntes propios sobre el capítulo 10 de OSTEP (*Multiprocessor Scheduling*). Fuente: https://ostep.org

Con varias CPUs, el scheduling deja de ser "¿quién ahora?" y pasa a ser "¿quién, ahora, y **dónde**?". El hardware manda: cachés y coherencia dictan las políticas.

## 10.1 El hardware que condiciona todo

**Cachés y localidad.** Cada CPU tiene cachés privadas (L1/L2) que explotan la localidad **temporal** (lo tocado hace poco se volverá a tocar) y **espacial** (lo contiguo a lo tocado también). Un hilo que corre un rato en una CPU deja allí su *estado caliente*.

**Coherencia de caché.** Con varias cachés privadas sobre una misma RAM aparece el riesgo de leer valores rancios: CPU-1 modifica X en su caché, el hilo migra a CPU-2 y lee la X vieja. El hardware lo resuelve con **protocolos de coherencia** (espionaje del bus / *bus snooping*, invalidación en escritura — familia MESI): cada caché vigila y cuando ve una escritura ajena a una línea que tiene, la invalida o actualiza.

**Coherencia ≠ sincronización.** Aunque el hardware mantenga coherentes las cachés, las secuencias de instrucciones sobre estructuras compartidas (p. ej. quitar de una lista enlazada desde dos CPUs) siguen necesitando **locks** — anticipo directo de la parte II del libro. Y con más CPUs, el lock que protege la cola del scheduler se convierte en cuello de botella: la sincronización no escala gratis.

**Afinidad de caché.** Consecuencia de todo lo anterior: **conviene volver a ejecutar cada hilo donde ya corrió**. Migrar un hilo cuesta invisiblemente caro (rellenar cachés y TLB desde cero).

## 10.2 SQMS: cola única

*Single-Queue Multiprocessor Scheduling*: reutilizar el scheduler monoprocesador con **una cola global**; cada CPU libre saca el siguiente trabajo.

- Pros: simple, balanceo de carga automático (nadie está ocioso si hay trabajo).
- Contras: (1) **escalabilidad** — todas las CPUs compiten por el lock de la cola; con muchas CPUs el sistema pasa más tiempo peleándose por el lock que trabajando; (2) **afinidad rota** — los trabajos van saltando de CPU en CPU según quién los saca de la cola, dejando sus cachés atrás. Se puede parchear con mecanismos de afinidad, pero contra natura.

## 10.3 MQMS: una cola por CPU

*Multi-Queue*: cada CPU tiene su propia cola y su scheduler local; los trabajos se asignan a una cola al llegar (p. ej. al azar o a la más corta).

- Pros: sin lock global (escala linealmente), afinidad perfecta por defecto (cada trabajo se queda en su CPU).
- Contra: **desbalanceo de carga**. Si a una CPU se le acaban los trabajos (o le tocan los cortos), queda ociosa o injustamente rápida mientras otra va saturada.

La solución al desbalanceo es la **migración**: mover trabajos entre colas. ¿Y cómo decidir cuándo? La técnica estándar es el **robo de trabajo (work stealing)**: una cola con poco trabajo mira *de vez en cuando* la de otra CPU y, si está notablemente más llena, le roba trabajos. La frecuencia del vistazo es el compromiso: mirar mucho recrea el problema de escalabilidad de SQMS; mirar poco deja desbalanceos. Otra constante vudú.

## 10.4 Los schedulers reales de Linux

El capítulo cierra constatando que no hubo consenso: Linux llegó a tener tres enfoques — **O(1)** (multi-cola, prioridades, estilo MLFQ), **CFS** (multi-cola, proporcional determinista) y **BFS/MuQSS** (¡cola única!, proporcional tipo EEVDF, popular en escritorios por su latencia). Los tres funcionaron: la ingeniería de scheduling es de compromisos, no de teoremas.

## Ampliación (no está en OSTEP)

- El scheduler actual de Linux (CFS/EEVDF) es MQMS con robo de trabajo estructurado en **dominios de scheduling** que calcan la jerarquía física: hermanos SMT → núcleos de la misma L3 → mismo nodo NUMA → toda la máquina. El balanceo es más reticente cuanto más "lejos" (más caro en cachés) esté el destino. Ver `/sys/kernel/debug/sched/domains/`.
- **NUMA** añade otra capa: migrar un hilo a otro nodo lo aleja de su *memoria*, no solo de sus cachés. `numactl`, AutoNUMA (migración automática de páginas hacia el hilo) y las políticas de primer toque son la respuesta de Linux.
- El paper *The Linux Scheduler: a Decade of Wasted Cores* (Lozi et al., EuroSys 2016) documenta bugs reales de balanceo que dejaban núcleos ociosos con trabajo pendiente durante años — lectura humillante y sana: ni el scheduler más usado del mundo está "terminado".
- El robo de trabajo es también el corazón de los runtimes N:M (goroutines de Go, Tokio de Rust, ForkJoinPool de Java): el mismo algoritmo un nivel más arriba (ver `../virtualizacion.md` §2.4).
- **Core scheduling** (Linux 5.14+): restricción de seguridad post-Spectre — solo hilos del mismo grupo de confianza pueden compartir los dos hermanos SMT de un núcleo (mitiga los canales laterales entre hilos hermanos).

## Referencias

- OSTEP cap. 10.
- Lozi et al., *The Linux Scheduler: a Decade of Wasted Cores*, EuroSys 2016.
- Blumofe & Leiserson, *Scheduling multithreaded computations by work stealing*, JACM 1999 (la teoría del work stealing, vía Cilk).
- Sorin, Hill & Wood, *A Primer on Memory Consistency and Cache Coherence* — para MESI y coherencia en serio.
- `Documentation/scheduler/sched-domains.rst` del kernel.
