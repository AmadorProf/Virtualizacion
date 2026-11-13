# 19 · TLB: traducciones rápidas

> Apuntes propios sobre el capítulo 19 de OSTEP (*Paging: Faster Translations (TLBs)*). Fuente: https://ostep.org

## 19.1 El problema y la solución

La paginación ingenua duplica los accesos a memoria (cada load/store necesita leer antes su PTE). La respuesta del hardware: la **TLB** (Translation Lookaside Buffer), una caché pequeña y rapidísima dentro de la MMU que guarda traducciones recientes VPN→PFN (+ bits de protección).

Algoritmo por acceso:

```
VPN en la TLB?
 ├─ sí (hit): física = PFN + offset  → coste ~0
 └─ no (miss): leer la tabla de páginas (page walk),
               insertar en TLB, reintentar
```

Como toda caché, funciona por **localidad**: espacial (recorrer un array toca la misma página muchas veces seguidas: 1 miss por página, luego hits) y temporal (lo traducido hace poco se vuelve a traducir). Tamaño típico: decenas–cientos de entradas por nivel; hit rate normal >99%. La TLB es, sin exagerar, **lo que hace practicable la memoria virtual**.

## 19.2 Ejemplo canónico: recorrer un array

Array de 10 enteros que cruza 3 páginas → recorrido: miss, hit, hit, miss, hit, hit, hit, miss, hit, hit = 70% de hits solo por localidad espacial; la segunda pasada (si es pronto) es 100% hits por temporal. Moral: el **tamaño de página** amortigua (páginas grandes = menos misses) y los patrones de acceso importan (recorrer por columnas una matriz guardada por filas = miss constante).

## 19.3 ¿Quién resuelve el miss: hardware o SO?

- **Hardware (CISC, x86)**: la MMU conoce el formato de la tabla y hace el *page walk* sola (registro CR3 → recorrer niveles). Rápido y opaco.
- **Software (RISC clásico: MIPS, SPARC)**: el miss lanza una excepción y **el SO** busca la traducción y la instala en la TLB con instrucciones privilegiadas. Más lento pero flexible: el SO puede usar cualquier estructura de datos como "tabla". Detalles delicados: el manejador debe estar en memoria *no paginada* (o de traducción fija) para no provocar misses recursivos infinitos.

## 19.4 Contenido y gestión de la TLB

Una entrada: VPN | PFN | válido | protección | ASID | sucia… Suele ser **totalmente asociativa** (una VPN puede estar en cualquier hueco; se busca en paralelo).

**El problema del cambio de contexto**: las traducciones son *por proceso*; las del proceso saliente son veneno para el entrante. Opciones:

- **Flush**: vaciar la TLB al cambiar (escribiendo CR3 en x86 clásico). Correcto y brutal: cada cambio de contexto arranca con TLB fría — gran parte del coste indirecto del context switch.
- **ASID** (Address Space ID): etiquetar cada entrada con un identificador de proceso; conviven traducciones de varios procesos y no hay flush. Lo tenían los RISC; x86 lo adoptó tarde como **PCID** (2010, y de verdad usado desde las mitigaciones Meltdown/KPTI 2018).

**Política de reemplazo** (TLB llena): LRU o aleatoria — aleatoria evita el caso patológico de un bucle sobre n+1 páginas con TLB de n (donde LRU falla el 100%).

## 19.5 Cuándo la TLB no basta

Programas cuyo *working set* de páginas excede la cobertura de la TLB (**TLB coverage**) pagan misses constantes — típico en bases de datos y estructuras aleatorias grandes. Remedios: **huge pages** (una entrada cubre 2 MiB/1 GiB: cobertura ×512/×262144), y diseño *cache/TLB-conscious* de estructuras. También hay costes de coherencia: la TLB es una caché de acceso físico dentro del pipeline; en multiprocesador, cambiar una PTE exige invalidar las TLBs de *otras* CPUs (**TLB shootdown**, con IPIs — caro).

## Ampliación (no está en OSTEP)

- Jerarquía real (x86 moderno): L1-ITLB + L1-DTLB (~64–128 entradas) + L2-STLB compartida (~1536–3072) — la TLB también es multinivel. Medir misses: `perf stat -e dTLB-load-misses,iTLB-load-misses ./programa`.
- **Huge pages en la práctica**: transparentes (THP, el kernel promociona 4 KiB→2 MiB solo; a veces con jitter indeseado — las bases de datos suelen pedir desactivarlo) o explícitas (`hugetlbfs`, reservadas al arranque; estándar en hipervisores y DPDK).
- En virtualización, cada miss de TLB cuesta el **doble de page walk** (tablas del guest × EPT: hasta 24 accesos — ver `../virtualizacion.md` §5.5): la TLB importa aún más dentro de una VM.
- Los page walkers modernos cachean niveles intermedios de la tabla (paging-structure caches) — caché para acelerar el relleno de otra caché.

## Referencias

- OSTEP cap. 19.
- Clark & Emer, *Performance of the VAX-11/780 Translation Buffer*, ACM TOCS 1985 — la medición clásica.
- Bhattacharjee & Lustig, *Architectural and OS Support for Virtual Memory* (Morgan & Claypool, 2017) — TLBs modernas a fondo.
- Intel SDM vol. 3, cap. 4 (paging) para PCID e invalidaciones.
