# 17 · Gestión del espacio libre

> Apuntes propios sobre el capítulo 17 de OSTEP (*Free-Space Management*). Fuente: https://ostep.org

## 17.1 El problema

Gestionar memoria libre es trivial con unidades **fijas** (una lista de páginas iguales: coge una cualquiera). Se vuelve difícil con unidades de **tamaño variable**: es el caso de `malloc`/`free` dentro del heap de un proceso, y del SO gestionando segmentos (cap. 16). El enemigo: la **fragmentación externa** — espacio libre total suficiente, pero picado en trozos donde no cabe la petición.

Condiciones del juego para un asignador tipo malloc: `free(ptr)` no recibe el tamaño (hay que apañárselas), no se puede compactar (los punteros entregados son sagrados: no puedes mover memoria en uso), y la región gestionada es una (creciente con `sbrk`/`mmap`).

## 17.2 Mecanismos de base

**Free list.** Estructura que enumera los huecos libres: lista enlazada (construida *dentro* de la propia memoria libre: cada hueco guarda su tamaño y el puntero al siguiente).

**División (splitting).** Petición de 1 byte y hueco de 10 → se parte: 1 se entrega, 9 quedan en la lista.

**Fusión (coalescing).** Al liberar, si el bloque es adyacente a huecos libres, se fusionan en uno mayor. Sin fusión, tras muchas liberaciones la lista degenera en confeti aunque *todo* esté libre.

**Cabeceras.** Cómo sabe `free()` cuánto liberar: `malloc` reserva unos bytes extra *antes* del puntero devuelto — una cabecera con el tamaño (y típicamente un número mágico de integridad). `free(ptr)` mira en `ptr - sizeof(header)`. Por eso pedir N bytes consume N + cabecera.

**Crecer el heap.** Lista vacía → pedir más al SO (`sbrk`/`mmap`) y añadirlo como hueco.

## 17.3 Políticas clásicas

¿Qué hueco elegir para una petición?

| Política | Idea | Pros/contras |
|---|---|---|
| **Best fit** | El hueco más pequeño que quepa | Desperdicia poco por decisión… pero genera astillas diminutas inútiles; y buscar cuesta (recorrido completo) |
| **Worst fit** | El hueco más grande | Intención: dejar restos grandes y útiles. En la práctica, de lo peor: fragmenta y busca mucho |
| **First fit** | El primero que quepa | Rápido; tiende a acumular astillas al principio de la lista |
| **Next fit** | Como first, pero desde donde quedó la última búsqueda | Reparte las astillas por toda la lista; rendimiento ~first fit |

Ninguna gana siempre: depende de la carga (tamaños y patrón de peticiones). Lección del capítulo: **sin conocer la carga, solo hay compromisos**.

## 17.4 Enfoques más elaborados

- **Listas segregadas**: listas separadas por tamaños populares. Caso extremo y célebre: el **slab allocator** (Bonwick, Solaris 1994) para el kernel — cachés de objetos del mismo tipo (inodos, locks) *preinicializados*: asignar = sacar de la caché, liberar = devolver sin destruir. Linux lo usa (SLUB) para todos los objetos del kernel.
- **Buddy allocator**: la memoria se parte recursivamente en mitades (potencias de 2). Asignar = bajar dividiendo hasta el tamaño justo; liberar = si tu "buddy" (la otra mitad) está libre, fusionar y subir recursivamente. Fusión elegantísima (el buddy se encuentra con un XOR de la dirección); fragmentación interna por el redondeo a potencia de 2. **Es el asignador de páginas físicas de Linux** (`/proc/buddyinfo`).
- **Escalabilidad**: los asignadores modernos atacan además la velocidad y el multihilo — búsquedas O(1) con árboles/índices y arenas por hilo para no serializar en un lock global.

## Ampliación (no está en OSTEP)

- El malloc real de glibc (**ptmalloc2**): bins segregados por tamaño + arenas por hilo + umbral mmap para peticiones grandes. Los de alto rendimiento (**jemalloc**, **tcmalloc**, **mimalloc**) refinan lo mismo: clases de tamaño, caches por hilo sin locks, devolución de memoria al SO por rangos (`madvise(MADV_DONTNEED)`).
- Robson (1971) probó **cotas teóricas**: cualquier asignador sin mover memoria puede necesitar ~log(ratio de tamaños) veces más memoria que la óptima — la fragmentación externa es matemáticamente inevitable, no un fallo de ingenio.
- Los GC compactadores (Java, .NET) esquivan el teorema **moviendo** objetos (pueden, porque controlan todos los punteros): la respuesta del otro lado del espejo.

## Referencias

- OSTEP cap. 17.
- Wilson et al., *Dynamic Storage Allocation: A Survey and Critical Review*, IWMM 1995 — el survey definitivo.
- Bonwick, *The Slab Allocator*, USENIX 1994.
- Knuth, *TAOCP vol. 1* §2.5 (buddy system).
- Robson, *An estimate of the store size necessary for dynamic storage allocation*, JACM 1971.
