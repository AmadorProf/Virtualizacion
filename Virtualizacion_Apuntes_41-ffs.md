# 41 · Localidad: el Fast File System (FFS)

> Apuntes propios sobre el capítulo 41 de OSTEP (*Locality and The Fast File System*). Fuente: https://ostep.org

## 41.1 El problema: el FS viejo trataba el disco como RAM

El FS original de Unix (layout simple: superbloque, inodes, datos) rendía terriblemente (~2% del ancho de banda del disco) porque ignoraba la mecánica: inodes lejos de sus datos (seek por cada acceso), ficheros fragmentados con el uso, bloques lógicamente consecutivos separados físicamente. La abstracción "array de bloques" era demasiado limpia.

**FFS** (Berkeley, McKusick et al., 1984) inaugura el principio que definió una era: **el FS debe conocer y explotar el dispositivo** ("disk awareness"), manteniendo la misma API — las mejoras van por dentro.

## 41.2 Grupos de cilindros: divide y acerca

FFS parte el disco en **grupos de cilindros** (hoy, "grupos de bloques" sobre direcciones lógicas), cada uno con su mini-copia de todo: superbloque (réplica, para recuperación), bitmaps propios, inodes propios y datos propios.

Políticas de colocación ("mantén cerca lo relacionado"):

- **Ficheros junto a su inode y su directorio**: los ficheros de un mismo directorio, en el mismo grupo (se abren juntos: `ls`, compilaciones).
- **Directorios**: en el grupo con pocos directorios y muchos inodes libres (equilibrar).
- **Excepción para ficheros grandes**: un fichero enorme reventaría la localidad de su grupo → se trocea en porciones repartidas entre grupos. El coste del seek entre porciones se **amortiza** con porciones suficientemente grandes (el análisis "chunking" del capítulo: para pagar solo un x% de peaje, porciones proporcionalmente mayores).

La justificación empírica: los estudios de tráfico real muestran localidad de namespace — lo que se accede junto está cerca en el árbol.

## 41.3 Detalles que hicieron a FFS usable

- **Bloques grandes + fragmentos**: bloques de 4 KiB (mejor throughput) desperdiciarían espacio con tantos ficheros pequeños → sub-bloques ("fragmentos" de 512 B) para las colas de los ficheros.
- **Parametrización de la rotación** (histórico): colocar los sectores consecutivos desfasados para no perder una vuelta entre lecturas — hoy irrelevante (la caché de pista del disco lo absorbe).
- Y las mejoras de **usabilidad** que quedaron para siempre: nombres largos de fichero, **symlinks**, y `rename` atómico — el capítulo subraya que hacer un sistema *usable* importó tanto como hacerlo rápido.

## Ampliación (no está en OSTEP)

- La descendencia es directa: **ext2/3/4 son FFS** en espíritu y en estructura (block groups, mismas políticas de colocación); XFS reinventó la idea con "allocation groups" pensados además para paralelismo.
- En SSDs la localidad *rotacional* murió, pero la de **bloque de borrado** y la de working set siguen importando (cap. 44); y la colocación por afinidad renace en los FS para SMR/ZNS (zonas append-only).
- La técnica de evaluación del capítulo — inventar cargas sintéticas que expongan una política (p. ej. "muchos dirs pequeños" vs "un fichero gigante") — es reutilizable: así se comparan FS hoy (fio, filebench).

## Referencias

- OSTEP cap. 41.
- McKusick, Joy, Leffler & Fabry, *A Fast File System for UNIX*, ACM TOCS 1984.
- McKusick & Neville-Neil, *The Design and Implementation of the FreeBSD Operating System* — FFS moderno (UFS2).
- `Documentation/filesystems/ext4/blockgroup.rst`.
