# 22 · Más allá de la memoria física: políticas de reemplazo

> Apuntes propios sobre el capítulo 22 de OSTEP (*Beyond Physical Memory: Policies*). Fuente: https://ostep.org

## 22.1 El problema y su métrica

Con la RAM llena, ¿**qué página expulsar**? La RAM funciona como una caché de las páginas del sistema; la métrica es el **tiempo medio de acceso**:

AMAT = T_RAM + P_miss × T_disco

Con T_disco ~10 ms y T_RAM ~100 ns, la diferencia entre 90% y 99,9% de hit rate es enorme: **cada décima de hit rate cuenta**, porque el coste del miss es monstruoso. Toda la discusión de políticas es "cómo arañar hits".

## 22.2 El óptimo como vara de medir

Política óptima (Belady, MIN): expulsar la página que se usará **más tarde en el futuro**. Imposible de implementar (requiere el futuro), pero imprescindible como **cota superior** en simulaciones: si tu política logra el 80% y el óptimo el 82%, estás fino; si el óptimo da 95%, tienes margen.

## 22.3 Políticas simples

- **FIFO**: expulsa la más antigua en llegar. Simple; ignora por completo el uso ("llegó primero" no implica "no se usa"). Sufre la **anomalía de Belady**: con ciertos patrones, *más* memoria da *más* fallos — FIFO no tiene la propiedad de inclusión de pila.
- **Aleatoria**: expulsa una al azar. Sorprendentemente decente (a veces roza a FIFO o mejor); sin estados patológicos persistentes.

## 22.4 Usar el pasado: LRU

Principio de **localidad**: lo usado recientemente se usará pronto otra vez. **LRU** (Least Recently Used) expulsa la página menos recientemente usada. En los patrones con localidad (bucles, working sets) arrasa a FIFO/Random; en accesos puramente aleatorios, todas empatan; y en el patrón trampa (bucle mayor que la memoria) LRU falla el 100% mientras Random se defiende. LRU además es "algoritmo de pila": más memoria nunca empeora.

Los tres regímenes de carga que usa el capítulo en sus simulaciones — sin localidad, 80/20 (80% de accesos al 20% caliente), bucle secuencial — son un kit mental reutilizable para evaluar cualquier caché.

## 22.5 LRU de verdad es caro → aproximarlo

LRU exacto exigiría actualizar una estructura ordenada **en cada acceso a memoria** — inviable. La solución práctica usa el **bit de accedida (A)** que el hardware pone solo (cap. 18):

**Algoritmo del reloj (clock)**: las páginas en un círculo con una manecilla. Para expulsar: mira la página bajo la manecilla — si A=1, ponlo a 0 y avanza (tuvo uso reciente: perdonada una vuelta); si A=0, expúlsala. Aproxima LRU con coste mínimo y es, con variantes, lo que usan los SO reales.

Mejora estándar: preferir expulsar páginas **limpias** (D=0, gratis) sobre sucias (D=1, hay que escribirlas) — reloj con dos bits (A, D).

## 22.6 Otras políticas del sistema completo

- **Prefetching**: traer la página N+1 al traer la N (secuencialidad).
- **Clustering** de escrituras: agrupar expulsiones sucias en una escritura grande.
- **Thrashing** (hiperpaginación): demanda total de working sets > RAM → el sistema solo pagina, no computa. Respuestas: control de admisión (correr menos procesos a la vez — "mejor hacer bien pocas cosas que mal todas") o la brutal actual: el **OOM killer**. Concepto teórico asociado: el *working set* de Denning.

## Ampliación (no está en OSTEP)

- Linux real: no un reloj puro sino **dos listas (activa/inactiva)** por tipo (anónimas/fichero) con promoción por segundo toque; desde 6.1, **MGLRU** (Multi-Generational LRU, de Google/Android): generaciones de edad más finas, mejor con working sets grandes.
- El área sigue viva en cachés de aplicación: **ARC** (IBM, usado por ZFS — equilibrio recencia/frecuencia adaptativo), **2Q**, **TinyLFU/W-TinyLFU** (Caffeine, la caché de referencia en Java), **S3-FIFO** (2023 — colas FIFO simples batiendo a LRU en trazas web). La lección de OSTEP (compara siempre contra óptimo y contra Random) se les aplica igual.
- La anomalía de Belady reaparece donde menos se espera: cachés de CPU con políticas no-stack y sistemas con capas de caché apiladas.

## Referencias

- OSTEP cap. 22.
- Belady, *A study of replacement algorithms*, IBM Systems Journal 1966.
- Denning, *The Working Set Model for Program Behavior*, SOSP 1967.
- Corbató, *A Paging Experiment with the Multics System*, 1968 (el clock).
- Megiddo & Modha, *ARC: A Self-Tuning, Low Overhead Replacement Cache*, FAST 2003.
- `Documentation/admin-guide/mm/multigen_lru.rst` del kernel.
