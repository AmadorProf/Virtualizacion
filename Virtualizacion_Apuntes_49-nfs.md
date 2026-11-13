# 49 · NFS: el sistema de ficheros en red de Sun

> Apuntes propios sobre el capítulo 49 de OSTEP (*Sun's Network File System (NFS)*). Fuente: https://ostep.org

## 49.1 Qué es un FS distribuido y por qué

Clientes que acceden por red a ficheros almacenados en un **servidor** central, a través de la API POSIX normal — el FS distribuido es transparente para las aplicaciones (el mismo `open`/`read`/`write`). Ventajas: datos compartidos entre máquinas, administración y backup centralizados, acceso desde cualquier puesto.

NFS (Sun, años 80) no fue un sistema sino un **protocolo abierto** — por eso hubo implementaciones de muchos fabricantes y por eso ganó. El objetivo de diseño dominante (NFSv2, el que estudia el capítulo): **recuperación simple ante fallos del servidor** — el servidor cae y rearranca, y los clientes deben seguir como si nada.

## 49.2 La clave: un servidor sin estado (stateless)

Decisión central de NFSv2: el servidor **no recuerda nada** de sus clientes — ni qué ficheros tienen abiertos, ni offsets, ni cachés ajenas. Cada petición lleva **toda** la información necesaria para resolverla por sí sola:

- Nada de "descriptores" del lado servidor: el protocolo usa **file handles** (volumen + nº de inode + generación) que el cliente guarda y presenta en cada operación.
- Operaciones autocontenidas: `read(handle, offset, count)` — el offset lo lleva el cliente (a diferencia del `read()` local, donde el offset vive en el kernel).
- El `open()` local se traduce en `LOOKUP` (nombre → handle); los `read/write` locales, en READs/WRITEs con handle+offset explícitos.

¿Por qué? Recuperación gratis: si el servidor rearranca (o simplemente iba lento), el cliente **reintenta la misma petición** y funciona — el servidor no necesitaba recordar nada para atenderla. Un servidor con estado tendría que reconstruir conversaciones (¿qué tenía abierto quién?) con protocolos de recuperación complejos.

Pieza que lo completa: reintentar exige que las operaciones sean **idempotentes** — LOOKUP y READ lo son trivialmente; WRITE con offset explícito también (escribir dos veces lo mismo en el mismo sitio = una vez). Algunas no lo son del todo (mkdir que reintentado devuelve "ya existe") — las esquinas imperfectas del diseño.

## 49.3 Caching: donde muere la sencillez

Sin caché, cada operación cruza la red → los clientes cachean datos y metadatos (y bufferizan escrituras). Aparece **el problema de consistencia de cachés**: el cliente A cachea una versión, B escribe otra — ¿cuándo ve A el cambio?

Solución (pragmática) de NFSv2, dos piezas:

- **Flush-on-close**: al cerrar un fichero, el cliente empuja todas sus escrituras al servidor → el siguiente que lo abra desde otra máquina verá la versión completa.
- **Validación con atributos**: antes de usar contenido cacheado, el cliente pregunta al servidor si el fichero cambió (GETATTR, comparando tiempos de modificación). Para no ahogar al servidor a GETATTRs, los atributos también se cachean unos segundos (~3 s) → **ventana de inconsistencia**: puedes leer contenido rancio durante esos segundos. "Suficientemente bueno" para el uso típico; inaceptable para otros usos — y así quedó: la consistencia de NFS es célebremente laxa.

Del lado servidor: las escrituras deben estar **en disco (o NVRAM) antes de responder OK** — si el servidor confirmara desde su RAM y cayera, el cliente (que ya no reintentará: le dijeron OK) perdería datos silenciosamente. Rendimiento de escritura = el precio de la honestidad; los vendedores lo resolvieron con NVRAM (y así nació NetApp, cuya historia arranca aquí).

## Ampliación (no está en OSTEP)

- Evolución: **NFSv3** (1995: escrituras asíncronas + COMMIT, mejores atributos); **NFSv4** (2003): el giro copernicano — **con estado** (opens, delegaciones de caché, locks integrados), compuestos (varias ops por RTT), seguridad seria (Kerberos). La industria acabó aceptando que el estado valía su complejidad. **pNFS** (v4.1) separa metadatos de datos para escalar.
- NFS sigue omnipresente (NAS corporativo, EFS de AWS es NFS gestionado); su competidor histórico de mundo Windows es SMB/CIFS.
- La lección transversal del capítulo — *diseña el protocolo alrededor de la recuperación, no como añadido* — es hoy dogma de sistemas cloud (idempotencia + reintentos = la mitad de la fiabilidad de cualquier API moderna).

## Referencias

- OSTEP cap. 49.
- Sandberg et al., *Design and Implementation of the Sun Network Filesystem*, USENIX 1985.
- Callaghan, *NFS Illustrated* — el protocolo con lupa.
- RFC 1813 (NFSv3), RFC 7530 (NFSv4).
