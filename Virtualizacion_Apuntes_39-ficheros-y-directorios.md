# 39 · Interludio: ficheros y directorios

> Apuntes propios sobre el capítulo 39 de OSTEP (*Interlude: Files and Directories*). Fuente: https://ostep.org

## 39.1 Las dos abstracciones

- **Fichero**: array lineal de bytes con un nombre de bajo nivel — su **inode number**. El FS no sabe ni le importa qué contiene (contrato "solo bytes").
- **Directorio**: fichero especial cuyo contenido es una lista de pares (nombre legible → inode). Directorios dentro de directorios → el **árbol** con raíz `/` y rutas como identificadores.

Unix lleva la abstracción más lejos: casi todo se nombra bajo el árbol (dispositivos en `/dev`, estado del kernel en `/proc`) — la unificación como principio de diseño.

## 39.2 La API esencial

- `open(ruta, flags, modo)` devuelve un **descriptor de fichero** (entero por-proceso): la capacidad para operar sobre el fichero. `O_CREAT|O_WRONLY|O_TRUNC` etc.
- `read(fd, buf, n)` / `write(fd, buf, n)`: secuenciales respecto al **offset** implícito que cada "open file" mantiene; `lseek()` lo mueve para acceso aleatorio (no mueve ningún brazo de disco: solo cambia un entero).
- Cada `open` crea una **entrada en la tabla de ficheros abiertos** del sistema (offset + modo + refcount). `fork()` **comparte** la entrada (offset compartido entre padre e hijo — ¡y refcount!); `dup()/dup2()` también — la base de las redirecciones del shell (cap. 5).
- `fsync(fd)`: fuerza a disco lo escrito (write solo garantiza llegar a la caché del SO). Sutileza importante: a veces hace falta *fsync del directorio* también (si el fichero es nuevo, su entrada de directorio debe persistir). Los que escriben bases de datos viven de estos detalles.
- Metadatos: `stat()` → tamaño, inode, permisos, tiempos, nº de enlaces.
- Nombres: `rename(viejo, nuevo)` — **atómico** respecto a fallos; el patrón universal de actualización segura: escribir `fichero.tmp`, fsync, y rename encima.
- Borrar: no hay syscall "delete": está `unlink()` — quita la entrada nombre→inode y decrementa el **contador de enlaces**; el inode y sus datos solo se liberan al llegar a 0 (y sin descriptores abiertos).

## 39.3 Enlaces duros y simbólicos

- **Hard link** (`ln`): otra entrada de directorio apuntando al **mismo inode** — dos nombres del mismo fichero, indistinguibles entre sí. Limitaciones: mismo FS, no a directorios (evitar ciclos).
- **Symlink** (`ln -s`): un fichero especial que contiene una **ruta** como texto. Cruza FS, apunta a lo que sea… y puede quedar **colgando** (dangling) si el destino desaparece.

## 39.4 Directorios y montaje

- API de directorios: `mkdir`, `opendir/readdir/closedir`, `rmdir` (solo vacíos). Toda dir nace con `.` y `..`.
- **Montaje** (`mount`): pegar el árbol de un FS sobre un directorio de otro → **un solo árbol de nombres** hecho de muchos FS (disco, USB, NFS, tmpfs, proc…). `mount` sin args lo lista.

## 39.5 Permisos

Bits clásicos rwx × (dueño, grupo, otros) — `chmod 640`. En directorios: x = poder atravesar. Más allá: ACLs para control fino. Aviso del capítulo con ejemplo real (la trampa del TOCTOU / los bits en sistemas compartidos): los permisos son la frontera de seguridad *dentro* del árbol compartido.

## Ampliación (no está en OSTEP)

- La cadena descriptor → tabla abierta → inode explica los clásicos: por qué un fichero borrado con procesos que lo tienen abierto sigue ocupando disco (`lsof | grep deleted`), y por qué dos `open()` independientes no comparten offset pero un `fork` sí.
- API moderna: `openat()` y familia *at (rutas relativas a un dirfd: cierra carreras TOCTOU), `O_TMPFILE` (fichero sin nombre desde el nacimiento), `statx()`, y `copy_file_range()`/reflinks (copias COW en XFS/Btrfs: `cp --reflink`).
- La atomicidad de `rename` es LA primitiva de fiabilidad en espacio de usuario: así actualizan sus ficheros SQLite, los editores serios y los gestores de paquetes. El detalle histórico doloroso: en ext4, rename sin fsync previo podía dar ficheros vacíos tras un crash (el escándalo "O_PONIES", 2009) — hoy ext4 lo mitiga con heurísticas.

## Referencias

- OSTEP cap. 39.
- Ritchie & Thompson, *The UNIX Time-Sharing System*, CACM 1974 — donde se definió casi todo esto.
- `man 2 open`, `man 2 rename`, `man 7 path_resolution`, `man 7 symlink`.
- Pillai et al., *All File Systems Are Not Created Equal*, OSDI 2014 — qué garantías da de verdad cada FS a las aplicaciones.
