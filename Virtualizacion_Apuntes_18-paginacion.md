# 18 · Paginación: introducción

> Apuntes propios sobre el capítulo 18 de OSTEP (*Paging: Introduction*). Fuente: https://ostep.org

## 18.1 La idea

Frente a trocear en piezas variables (segmentación, con su fragmentación externa), la paginación trocea **todo en piezas fijas**:

- El espacio **virtual** se divide en **páginas** (típicamente 4 KiB).
- La RAM **física** se divide en **marcos** (page frames) del mismo tamaño.
- Cualquier página virtual puede ir a **cualquier marco físico**: la asignación es trivial (una free list de marcos iguales) y la fragmentación externa desaparece.

El mapa página→marco de cada proceso es su **tabla de páginas**.

## 18.2 La traducción

Una dirección virtual se parte en dos: número de página virtual (**VPN**, bits altos) y desplazamiento (**offset**, bits bajos, intacto en la traducción).

```
 virtual:  [ VPN ][ offset ]
              │ tabla de páginas
              ▼
 física:   [ PFN ][ offset ]
```

Ejemplo mínimo: espacio de 64 bytes con páginas de 16 → 2 bits de VPN, 4 de offset. Real x86-64 con páginas de 4 KiB: 12 bits de offset; con 48 bits virtuales, 36 de VPN → 2^36 páginas posibles. Ahí está el problema (§18.4).

## 18.3 La entrada de tabla de páginas (PTE)

En su forma lineal más simple, la tabla es un array indexado por VPN; cada **PTE** contiene el número de marco (**PFN**) más bits de control:

- **P (presente)**: ¿está la página en RAM? (si no: swap, cap. 21).
- **R/W** y **U/S**: escribible; accesible desde modo usuario.
- **A (accedida)** y **D (sucia)**: la puso el hardware al leer/escribir — materia prima de las políticas de reemplazo (cap. 22).
- **NX/XD**: no ejecutable (defensa: pilas y heaps sin ejecución).
- Acceso inválido o sin permiso → excepción → el SO decide (típicamente `SIGSEGV`).

## 18.4 Los dos problemas de la paginación ingenua

**1. Espacio.** Tabla lineal de x86-64: 2^36 entradas × 8 B = **512 GiB por proceso**. Absurdo. (Incluso el ejemplo de 32 bits del libro: 2^20 × 4 B = 4 MiB por proceso, ya doloroso con cientos de procesos.) Solución: tablas multinivel (cap. 20).

**2. Tiempo.** Cada acceso del programa ahora requiere **un acceso extra a memoria** (leer la PTE) — la máquina iría a la mitad de velocidad o peor. Solución: cachear traducciones (TLB, cap. 19).

La paginación pura es, pues, un esqueleto correcto (sin fragmentación externa, protección por página, compartición fácil) al que le faltan sus dos órganos vitales.

## 18.5 Rastro de un acceso

Ejercicio mental que propone el capítulo y conviene interiorizar: un simple `array[i]++` en un bucle genera *por iteración*: 1 fetch de instrucción (acceso a la tabla + acceso al código), 1 load y 1 store (cada uno: acceso a tabla + acceso al dato). Sin TLB, **cada acceso lógico son dos físicos**. Visualizar este rastro es la mejor vacuna contra la sorpresa de "por qué importa tanto la TLB".

## Ampliación (no está en OSTEP)

- Tamaños de página hoy: 4 KiB estándar x86; ARM64 permite 4/16/64 KiB (Apple usa 16 KiB — menos entradas, mejor TLB, más fragmentación interna). **Huge pages** de 2 MiB/1 GiB: cap. 19 y `../virtualizacion.md` §3.2.
- La fragmentación que la paginación **no** elimina: la **interna** (media página desperdiciada al final de cada región). Con páginas de 4 KiB es despreciable; con huge pages de 1 GiB, no — de ahí que convivan tamaños.
- Ver las tablas en vivo: `/proc/<pid>/pagemap` expone (con permisos) el mapeo VPN→PFN real de cualquier proceso; la herramienta `page-types` del kernel lo decodifica.

## Referencias

- OSTEP cap. 18.
- Kilburn et al., *One-level storage system* (Atlas, 1962) — donde nació la paginación.
- Jacob & Mudge, *Virtual Memory in Contemporary Microprocessors*, IEEE Micro 1998.
