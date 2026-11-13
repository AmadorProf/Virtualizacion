# 32 · Bugs de concurrencia comunes

> Apuntes propios sobre el capítulo 32 de OSTEP (*Common Concurrency Problems*). Fuente: https://ostep.org

## 32.1 El mapa del territorio

El capítulo se apoya en un estudio empírico famoso (Lu et al., ASPLOS 2008: bugs reales de MySQL, Apache, Mozilla, OpenOffice) con un hallazgo que reordena prioridades: la **mayoría de los bugs de concurrencia no son deadlocks** — son bugs de *no-deadlock* (~2/3), y de esos, casi todos caen en solo **dos patrones**.

## 32.2 Bugs de no-deadlock

**a) Violación de atomicidad.** Dos accesos que el programador *asumía* indivisibles se entrelazan con otro hilo. Patrón típico: comprobar un puntero y usarlo después, con otro hilo poniéndolo a NULL en medio.

```c
// hilo 1                       // hilo 2
if (p->info != NULL)
                                p->info = NULL;
    usar(p->info);              // ¡boom!
```

Arreglo: un lock alrededor de la secuencia completa comprobación+uso (y en cualquier otro sitio que toque ese dato).

**b) Violación de orden.** El código asume que A ocurre antes que B, pero nada lo garantiza (p. ej. un hilo usa algo que otro hilo inicializa, y el scheduler ejecuta al usuario antes que al inicializador). Arreglo: **imponer el orden** con una CV (o semáforo iniciado a 0): el consumidor espera; el inicializador señala.

Los dos patrones cubren ~97% de los bugs no-deadlock del estudio. Moraleja práctica: cuando revises código concurrente, busca *secuencias que se creen atómicas* y *órdenes que se dan por supuestos*.

## 32.3 Deadlock: las cuatro condiciones

Un **interbloqueo** requiere simultáneamente (Coffman et al., 1971):

1. **Exclusión mutua**: recursos no compartibles (locks).
2. **Hold-and-wait**: retener recursos mientras se esperan otros.
3. **No expropiación**: no se pueden quitar por la fuerza.
4. **Espera circular**: un ciclo A→B→…→A de "espero lo que tienes".

Romper *cualquiera* de las cuatro previene el deadlock. Por qué ocurren en sistemas grandes: dependencias complejas entre módulos (el ejemplo clásico VFS↔FS), y la **encapsulación** jugando en contra — llamas a `f(a,b)` de una biblioteca que coge locks internos en un orden que no ves (el ejemplo de `addAll(v1, v2)` de Java: dos hilos llamándolo con argumentos cruzados se bloquean).

## 32.4 Prevención (romper una condición)

- **Espera circular** → **orden total (o parcial) de adquisición de locks**: documentar y respetar "siempre L1 antes que L2". Truco robusto cuando el orden es dinámico: adquirir por dirección de memoria (`if (l1 > l2) swap`). Es la técnica dominante en la práctica.
- **Hold-and-wait** → adquirir todos los locks de golpe bajo un meta-lock. Funciona pero mata la concurrencia y exige conocer los locks por adelantado; poco usado.
- **No expropiación** → `trylock` + soltar todo y reintentar si falla. Ojo con el **livelock** (dos hilos reintentando eternamente en fase; se mitiga con backoff aleatorio) y con deshacer trabajo a medias.
- **Exclusión mutua** → prescindir de locks: estructuras **lock-free** con CAS (insertar en lista con bucle de compare-and-swap). Potente y delicado.

## 32.5 Evitación y recuperación

- **Evitación por scheduling** (Banker's algorithm de Dijkstra como pariente teórico): si el sistema *conoce* qué locks usará cada hilo, puede planificar para que los conflictivos no corran a la vez. Requiere conocimiento previo y reduce paralelismo → nicho (embebidos).
- **Detectar y recuperar**: dejar que pase, detectarlo (grafo de esperas) y actuar — reiniciar, abortar una transacción. Es lo que hacen las **bases de datos** (deadlock detector + rollback de una transacción víctima), donde abortar-y-reintentar es natural. "No todo lo raro hay que prevenirlo si es raro y recuperable" (la cita de Tom West que usa el libro como aside).

## Ampliación (no está en OSTEP)

- Herramientas: **lockdep** del kernel Linux (valida en runtime el orden de adquisición y chilla ante inversiones *aunque el deadlock no llegue a darse*), TSan también detecta inversiones de orden de lock; en Java, los thread dumps detectan ciclos automáticamente.
- El estudio de Lu et al. también midió los *arreglos*: la mayoría de fixes de violaciones de atomicidad fueron... añadir un lock; y una fracción notable de los parches **introdujo nuevos bugs** — argumento empírico para primitivas de más alto nivel (canales, actores, STM) y para lenguajes que verifican (Rust elimina data races en compilación, aunque los deadlocks siguen siendo posibles: son un problema de *lógica*, no de memoria).
- MapReduce (que cita el capítulo) como recordatorio de la vía alternativa: diseñar el problema para **no compartir estado** — la prevención definitiva.

## Referencias

- OSTEP cap. 32.
- Lu, Park, Seo & Zhou, *Learning from Mistakes — A Comprehensive Study on Real World Concurrency Bug Characteristics*, ASPLOS 2008.
- Coffman, Elphick & Shoshani, *System Deadlocks*, Computing Surveys 1971.
- `Documentation/locking/lockdep-design.rst` del kernel.
