# 42 · Consistencia tras fallos: FSCK y journaling

> Apuntes propios sobre el capítulo 42 de OSTEP (*Crash Consistency: FSCK and Journaling*). Fuente: https://ostep.org

## 42.1 El problema

Una operación de FS toca **varias estructuras** (añadir un bloque a un fichero: bitmap de datos + inode + bloque de datos), pero el disco solo escribe **un sector atómicamente**, y el write buffering reordena y retrasa. Si se va la luz en medio, el disco queda con un **subconjunto arbitrario** de las escrituras aplicado.

Casos posibles del ejemplo (3 escrituras, cae la luz):

- Solo el dato → inofensivo (el FS ni lo ve: como si no hubiera pasado).
- Solo el inode → lee **basura** (apunta a un bloque nunca escrito) y además **inconsistencia** inode/bitmap.
- Solo el bitmap → **fuga de espacio** (bloque marcado usado, sin dueño).
- Inode+bitmap sin dato → consistente… con basura dentro.
- Etc.

Dos males distintos: **inconsistencia entre metadatos** y **pérdida/basura de datos**. El objetivo mínimo de los FS: garantizar la consistencia de metadatos; los datos son negociables (¡y ahí están los modos de journaling!).

## 42.2 Solución 1: FSCK — dejar que pase y arreglar después

El enfoque histórico: tras un crash, **fsck** recorre TODO el FS y repara inconsistencias — verifica superbloque, reconstruye bitmaps desde los inodes, comprueba estados de inodes, contadores de enlaces (huérfanos a `lost+found`), punteros duplicados, integridad de directorios…

Dos problemas fatales:

1. **No arregla todo**: puede dejar el FS *consistente* con datos basura (fsck solo entiende de metadatos).
2. **Es lentísimo**: escanear discos de TB tarda horas — "pagar el precio de escanear toda la casa por unas llaves perdidas en el dormitorio". Con los tamaños modernos, inviable como plan principal.

## 42.3 Solución 2: journaling (write-ahead logging)

Idea tomada de las bases de datos: **antes** de tocar las estructuras definitivas, escribir en un **diario** (región reservada) una nota con lo que vas a hacer. Si hay crash, el diario dice qué estaba a medias → recuperación = **releer el diario** (segundos), no escanear el disco.

**Data journaling** (todo al diario) — la forma didáctica:

1. Escribir al journal: TxBegin + [bitmap, inode, dato] …
2. … y el **commit** (TxEnd) *solo cuando lo anterior está en disco* — dos pasos separados por una barrera, porque el disco reordena y un TxEnd adelantado haría parecer válida una transacción corrupta (el aside célebre: discos que mienten sobre sus cachés).
3. **Checkpoint**: escribir las estructuras en su sitio definitivo.
4. Liberar el espacio del journal (es circular).

Recuperación: transacciones con commit → **redo** (reaplicarlas); sin commit → ignorar. Las escrituras se vuelven atómicas: o toda la transacción o nada.

**Metadata journaling** (ordered) — lo que se usa de verdad: escribir cada dato **dos veces** duele; en su lugar, solo los *metadatos* van al diario, y el **dato se escribe primero en su sitio definitivo** (regla de oro: el dato antes que el commit de sus metadatos → un inode nunca apunta a basura). Es el modo por defecto de ext4 (`data=ordered`).

Detalle fino: **reutilización de bloques** (un bloque que fue de directorio, liberado y reusado como dato puede ser machacado por un replay del diario) → registros de revocación.

## 42.4 Otras vías

- **Soft updates**: ordenar cuidadosamente *todas* las escrituras para que el disco nunca esté inconsistente (el dato antes que el inode que lo apunta…). Elegante, endiabladamente difícil de implementar (BSD lo hizo; casi nadie más).
- **Copy-on-write**: nunca sobrescribir; escribir versiones nuevas y conmutar la raíz atómicamente — el mundo de LFS/ZFS/Btrfs (cap. 43).
- **Optimistic crash consistency**: reducir los flushes con checksums en las transacciones (OptFS, del propio grupo de los autores).

## Ampliación (no está en OSTEP)

- Panorama actual: ext4 (journal, jbd2), XFS (journal de metadatos siempre), NTFS (ídem); Btrfs/ZFS/APFS (COW: sin journal clásico); f2fs (log-structured para flash). fsck sigue existiendo como *último recurso*, no como plan.
- Las **barreras de escritura** del paso 2 son hoy los flags FUA/flush de la capa de bloques; el journaling depende de que el hardware los respete — los discos/controladoras que mienten siguen causando corrupciones (y ZFS añade checksums precisamente por desconfianza, cap. 45).
- La lección de capas: el journaling del FS protege *metadatos del FS*; una aplicación que necesita atomicidad de *sus* datos debe construirla encima (fsync + rename, o su propio WAL — SQLite, Postgres). Referencia dolorosa: Pillai et al., OSDI 2014.

## Referencias

- OSTEP cap. 42.
- Hagmann, *Reimplementing the Cedar File System Using Logging and Group Commit*, SOSP 1987 — el journaling de metadatos original.
- Tweedie, *Journaling the Linux ext2fs Filesystem*, 1998 (ext3).
- Ganger & Patt, *Metadata Update Performance in File Systems*, OSDI 1994 (soft updates).
- Chidambaram et al., *Optimistic Crash Consistency*, SOSP 2013.
