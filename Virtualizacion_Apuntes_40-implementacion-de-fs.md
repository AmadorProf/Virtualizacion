# 40 · Implementación de un sistema de ficheros

> Apuntes propios sobre el capítulo 40 de OSTEP (*File System Implementation*). Fuente: https://ostep.org

## 40.1 El método: estructuras + rutas de acceso

Para entender cualquier FS, dos preguntas: ¿qué **estructuras** hay en disco? y ¿qué pasa (qué se lee/escribe) en cada **operación** (open, read, write…)? El capítulo construye un FS mínimo ("VSFS") al estilo del FS original de Unix.

## 40.2 Layout en disco

Partición de N bloques (bloque = 4 KiB):

```
| S | ib | db | I I I I I | D D D D D D D D ... |
 superbloque  bitmaps      tabla de inodes    bloques de datos
```

- **Superbloque**: metadatos del FS (tamaños, dónde empieza cada región, magic number). Se lee al montar.
- **Bitmaps** de inodes y de datos: qué está libre/ocupado.
- **Tabla de inodes**: array de inodes; el inode number es el índice (por eso localizar un inode es aritmética simple).
- **Bloques de datos**: la mayoría del disco.

## 40.3 El inode

La estructura por-fichero: tipo, tamaño, permisos, dueño, tiempos, contador de enlaces… y sobre todo **dónde están los datos**:

- **Punteros directos** (~12): bloques de datos directamente.
- **Puntero indirecto**: apunta a un bloque lleno de punteros (~1024 más).
- **Doble/triple indirecto**: árbol desbalanceado a favor de los ficheros pequeños — decisión respaldada por la realidad medida: **la mayoría de los ficheros son pequeños** (~2 KiB de mediana; la tabla de "verdades sobre ficheros" de Agrawal et al. que cita el capítulo).
- Alternativa: **extents** (rango contiguo: inicio+longitud) — más compacto, menos flexible; lo usan ext4, XFS, NTFS.

Los **directorios**: simplemente ficheros cuyo contenido son entradas (inode, longitud, nombre); borrar deja huecos (por eso las entradas llevan longitud de registro). FS reales usan estructuras mejores (árboles B en XFS/ext4-htree) para dirs enormes.

**Gestión de libres**: los bitmaps al asignar/liberar; con heurísticas (buscar secuencias contiguas, pre-asignación) para la contigüidad.

## 40.4 Rutas de acceso — el corazón del capítulo

**Leer `/foo/bar`**: cada nivel de la ruta cuesta lecturas — inode de `/` → datos de `/` (buscar "foo") → inode de foo → datos de foo (buscar "bar") → inode de bar → ya con permiso comprobado, `read()`: inode (localizar bloque) → bloque de datos → actualizar inode (atime). La longitud de la ruta multiplica las E/S.

**Crear y escribir es peor**: `create` toca bitmap de inodes (leer+escribir), datos y del directorio, inode nuevo y del directorio…; cada `write()` que asigna bloque: 5 E/S (bitmap ×2, inode ×2, dato). Una escritura "sencilla" son decenas de E/S. Conclusión: un FS sin cachés sería inutilizable.

## 40.5 Caching y buffering

- **Lecturas**: la **page cache** unificada (memoria dinámica compartida con la MV, no particiones fijas como antaño) absorbe la mayoría — el tráfico de rutas populares (inodes de `/`, `/usr`…) se sirve de RAM.
- **Escrituras**: el **write buffering** las retrasa (5–30 s): agrupa (el bitmap tocado por 2 ficheros se escribe una vez), planifica mejor, y a veces **evita** E/S (fichero temporal creado y borrado antes del flush: cero escrituras). El precio: un crash pierde lo no volcado → las aplicaciones que no pueden permitírselo usan `fsync()` o E/S directa (bases de datos). El eterno compromiso durabilidad/rendimiento.

## Ampliación (no está en OSTEP)

- Mapa a FS reales: ext4 = este diseño + extents + grupos de bloques (cap. 41) + journal (cap. 42) + htree; XFS = extents + B+trees por todas partes; Btrfs/ZFS = copy-on-write total (parientes del cap. 43).
- Explora en vivo: `dumpe2fs /dev/sdX` (superbloque y grupos), `debugfs` (inodes y entradas de dir a mano), `stat fichero` (inode, bloques), `df -i` (inodes agotables: sí, puedes quedarte sin inodes con disco libre).
- El coste de las rutas largas es real hoy: el kernel lo amortigua con la **dcache** (caché de dentry nombre→inode), una de las estructuras más calientes de Linux.

## Referencias

- OSTEP cap. 40.
- Agrawal et al., *A Five-Year Study of File-System Metadata*, FAST 2007.
- Bach, *The Design of the UNIX Operating System* — el FS clásico de Unix con todo detalle.
- `Documentation/filesystems/ext4/` del kernel.
