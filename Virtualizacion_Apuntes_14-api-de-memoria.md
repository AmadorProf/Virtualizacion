# 14 · Interludio: la API de memoria

> Apuntes propios sobre el capítulo 14 de OSTEP (*Interlude: Memory API*). Fuente: https://ostep.org

## 14.1 Dos tipos de memoria para el programador C

- **Pila (automática)**: `int x;` dentro de una función. El compilador la reserva al entrar y la libera al salir — cómodo, pero nada sobrevive al `return`.
- **Heap (dinámica)**: `malloc()`/`free()`. Vive hasta que la liberes explícitamente. Toda la potencia, todos los problemas.

## 14.2 La API

```c
void *malloc(size_t size);   // reserva size bytes (o NULL si falla)
void free(void *ptr);        // libera lo que devolvió malloc
void *calloc(size_t n, size_t sz);   // como malloc pero pone a cero
void *realloc(void *ptr, size_t sz); // redimensiona (puede mover)
```

Hábitos correctos:

- Pedir tamaños con `sizeof(tipo)`, no números mágicos: `malloc(10 * sizeof(int))`.
- Para cadenas: `malloc(strlen(s) + 1)` — el `+1` del terminador `\0` es el olvido clásico.
- `free()` recibe exactamente el puntero que devolvió `malloc` (no uno desplazado): el asignador guarda el tamaño en una cabecera junto al bloque (cap. 17).
- Ojo: `sizeof(puntero)` = 8, no el tamaño de lo apuntado — trampa habitual.

Importante: `malloc`/`free` **no son syscalls**, son funciones de biblioteca que gestionan un almacén propio; piden memoria al SO por debajo con `brk`/`sbrk` (mover el final del heap) o `mmap` (regiones anónimas). Nunca llames a `brk` tú mismo.

## 14.3 El catálogo de errores

Los errores de memoria en C son la fuente clásica de bugs y vulnerabilidades:

| Error | Qué pasa |
|---|---|
| Olvidar reservar (`strcpy` a puntero sin malloc) | Segfault o corrupción |
| Reservar de menos (**buffer overflow**) | Sobrescribes lo vecino; base de exploits históricos |
| Olvidar inicializar | Lees basura del heap (**uninitialized read**) |
| Olvidar liberar (**memory leak**) | El proceso engorda hasta agotar memoria; letal en servidores de larga vida |
| Liberar y seguir usando (**use-after-free / dangling pointer**) | Corrupción; hoy la vulnerabilidad más explotada |
| Liberar dos veces (**double free**) | Corrompe el asignador |
| `free()` de un puntero no-malloc | Ídem |

Matiz sobre los leaks: en un proceso **de vida corta** no son fatales — al morir el proceso, el SO recupera *toda* su memoria (las dos capas de gestión: asignador dentro del proceso, SO por debajo). En procesos de larga vida, sí lo son.

## 14.4 Herramientas

- **valgrind (memcheck)**: detecta leaks, use-after-free, lecturas sin inicializar. Lento (~20×) pero exhaustivo.
- **AddressSanitizer (ASan)**: `gcc -fsanitize=address` — instrumentación en compilación, ~2× de coste, detección inmediata. Y hermanos: LeakSanitizer, MSan (uninitialized), UBSan.

```bash
gcc -g -fsanitize=address programa.c && ./a.out   # ASan
valgrind --leak-check=full ./a.out                # valgrind
```

## Ampliación (no está en OSTEP)

- Asignadores reales: glibc usa **ptmalloc2** (arenas por hilo); alternativas de alto rendimiento: **jemalloc** (Facebook/FreeBSD), **tcmalloc** (Google), **mimalloc** (Microsoft). Cambiarlos es trivial (`LD_PRELOAD`) y en servidores multihilo puede dar mejoras de dos dígitos.
- Umbral mmap: glibc sirve peticiones grandes (≥128 KiB por defecto) con `mmap` directo, de modo que `free` las devuelve al SO de verdad; las pequeñas van al heap y rara vez vuelven al sistema (fragmentación).
- La respuesta moderna a este catálogo de errores es **mover la seguridad al lenguaje**: Rust (ownership/borrowing hace imposibles use-after-free y double-free en código seguro) o GC (Java, Go). ~70% de las vulnerabilidades graves de Microsoft y Chrome eran de memoria — de ahí la presión (informes MSRC 2019, Chromium Memory Safety).

## Referencias

- OSTEP cap. 14.
- `man 3 malloc`, `man 2 mmap`, `man 2 brk`.
- Novark & Berger, *DieHarder: Securing the heap*, CCS 2010 — asignadores y seguridad.
- Serebryany et al., *AddressSanitizer*, USENIX ATC 2012.
