# 43 · Sistemas de ficheros log-structured (LFS)

> Apuntes propios sobre el capítulo 43 de OSTEP (*Log-structured File Systems*). Fuente: https://ostep.org

## 43.1 La observación y la apuesta

Contexto (Berkeley, ~1991, Rosenblum & Ousterhout): la RAM crece → las **lecturas** las absorbe cada vez más la caché → el tráfico de disco lo dominan las **escrituras**; y el disco es rapidísimo en secuencial y pésimo en aleatorio (cap. 37). FFS, aun con localidad, sigue haciendo muchas escrituras cortas dispersas (inode, bitmap, dir, dato…).

La apuesta radical de LFS: **no sobrescribir nunca nada; escribir TODO (datos y metadatos) como un log secuencial infinito**. Cada escritura va al final del log; las versiones viejas quedan atrás como basura a recoger.

## 43.2 Cómo funciona

- **Buffering en segmentos**: escribir secuencialmente no basta si escribes poquitos (el disco gira entre una y otra); LFS acumula escrituras en RAM y vuelca **segmentos** grandes (~1 MiB) de un golpe — el tamaño se calcula para amortizar el posicionamiento (misma matemática de chunking del cap. 41).
- **¿Y cómo encuentras los inodes** si están regados por el log? Cadena de indirección: el **inode map** (imap) traduce nº de inode → su última posición en disco; el imap también se escribe en el log (junto a lo que modifica), y una **checkpoint region** fija (actualizada cada ~30 s) apunta a los trozos del imap. Lectura: CR → imap (cacheado en RAM) → inode → dato — tras el arranque, casi todo desde caché, igual que un FS normal.
- **Directorios**: guardan (nombre → nº de inode), no posiciones — el imap añade el nivel de indirección que evita el **problema de actualización recursiva** (si el dir guardara posiciones, cada movimiento de un inode obligaría a reescribir el dir, y el dir del dir…).

## 43.3 El precio: recolección de basura

Sobrescribir un fichero deja las versiones viejas en el log. LFS **limpia por segmentos**: lee M segmentos parcialmente vivos, compacta lo vivo en N < M segmentos nuevos al final del log, y libera los M. Para saber qué está vivo: cada segmento lleva un **segment summary** (qué inode/offset era cada bloque) — se comprueba contra el imap: si el inode ya no apunta ahí, es basura.

**Qué limpiar y cuándo**: la política importa — el paper propone limpiar preferentemente segmentos **fríos** (estables: liberar sus huecos renta a largo plazo) antes que calientes (se ensuciarán de nuevo enseguida). El coste del cleaning fue el flanco de las guerras académicas de los 90 (LFS vs FFS mejorado).

**Recuperación tras crash**: la CR se actualiza con doble copia + timestamps (atómica); luego **roll-forward**: releer el log desde la CR y recuperar lo escrito después — journaling llevado a su conclusión lógica: *todo el disco es el journal*.

## 43.4 El legado — la idea que ganó 30 años después

LFS como FS de disco no triunfó entonces, pero su principio — **convertir escrituras aleatorias en secuenciales + versionado inmutable + GC** — está en todas partes hoy:

- **SSDs**: la FTL de *toda* SSD es un LFS dentro del dispositivo (cap. 44) — el flash no puede sobrescribir, exactamente el escenario LFS.
- **FS copy-on-write**: ZFS, Btrfs, WAFL (NetApp), APFS — no sobrescribir + conmutar raíz atómica + snapshots gratis (las "versiones viejas" del log, convertidas en feature).
- **Bases de datos y almacenes**: **LSM-trees** (LevelDB/RocksDB, Cassandra, BigTable) son la misma idea aplicada a claves-valor: memtable → volcados secuenciales → compactación (=cleaning).
- **f2fs**: un LFS de verdad, en tu móvil Android, diseñado para flash.

## Ampliación (no está en OSTEP)

- El debate del cleaning se reeditó como **write amplification** de las SSDs y las compactaciones de LSM: la misma matemática (utilización de segmentos vs coste de limpiar) con otros nombres. La fórmula de coste del capítulo se reusa tal cual.
- El versionado inmutable como principio trascendió el almacenamiento: git (objetos inmutables + refs conmutadas atómicamente) es estructuralmente un LFS.

## Referencias

- OSTEP cap. 43.
- Rosenblum & Ousterhout, *The Design and Implementation of a Log-Structured File System*, ACM TOCS 1992.
- Seltzer et al., *An Implementation of a Log-Structured File System for UNIX*, USENIX 1993 (y la réplica de 1995: el debate).
- O'Neil et al., *The Log-Structured Merge-Tree*, Acta Informatica 1996.
- Lee et al., *F2FS: A New File System for Flash Storage*, FAST 2015.
