# 45 · Integridad y protección de datos

> Apuntes propios sobre el capítulo 45 de OSTEP (*Data Integrity and Protection*). Fuente: https://ostep.org

## 45.1 Más allá del fail-stop

El RAID (cap. 38) asumía discos que fallan **limpiamente** (funcionan o están muertos). La realidad moderna es peor: discos que fallan **parcialmente y en silencio**. Dos modos principales:

- **Latent Sector Errors (LSE)**: un sector concreto ilegible (daño físico, ECC interno incapaz de corregir). Al menos el disco **avisa** (error de lectura explícito). Los estudios de campo citados (Bairavasundaram et al.) los encuentran en un porcentaje relevante de discos, más en los baratos.
- **Corrupción silenciosa**: el disco devuelve **datos incorrectos sin error** — escrituras que no se hicieron (*lost writes*), escrituras en el sitio equivocado (*misdirected writes*, por bugs de firmware), bits podridos que el ECC no ve. El modo más peligroso: no hay señal.

## 45.2 Contra LSE: redundancia

Los LSE se resuelven con lo del cap. 38: al fallar la lectura, reconstruir desde espejo/paridad. Consecuencia de diseño ya estándar: **RAID-6/doble paridad** — porque el escenario "muere un disco entero Y aparece un LSE durante la reconstrucción" pasó de teórico a estadísticamente esperable con discos grandes. También: *scrubbing* periódico (leer todo el disco en background para cazar LSEs antes de necesitarlos).

## 45.3 Contra la corrupción silenciosa: checksums

Si el disco no avisa, el sistema debe **verificar**: guardar junto a cada bloque un **checksum** (huella pequeña de su contenido) y comprobarlo en cada lectura.

- Funciones: XOR (débil), suma/Fletcher, **CRC** — el compromiso coste/detección habitual; colisiones posibles pero improbables.
- **Dónde guardarlo**: sectores de 520 B (formato con área extra, discos enterprise) o agrupar checksums en sectores aparte (más E/S).
- Detección al leer: checksum no cuadra → corrupción detectada → recuperar de la redundancia (o error honesto si no hay copia). Convertir corrupción *silenciosa* en *ruidosa* es el 90% de la batalla.

**Casos que el checksum simple no caza:**

- **Misdirected writes**: el bloque escrito en el LBA equivocado tiene un checksum *válido* (de sí mismo). Solución: añadir al checksum el **ID físico** (disco + LBA esperado) — si lees el bloque D10 y dice ser D12, algo falla.
- **Lost writes**: la escritura nueva nunca llegó; lo viejo (bloque Y checksum viejos, consistentes entre sí) pasa todas las pruebas. Soluciones: verificar tras escribir (caro) o checksums referenciales (el inode guarda el checksum de sus bloques — ZFS: el árbol entero se auto-verifica).

**Cuándo comprobar**: en cada lectura + **scrubbing** periódico (la corrupción en datos fríos, sin scrub, se descubre cuando ya no hay copia sana).

## 45.4 Coste

Espacio: ~0.2% (8 B por 4 KiB) — despreciable. CPU: calcular el checksum en cada read/write — moderado (y las CPUs traen instrucciones CRC32). E/S extra: según el layout de los checksums. Barato para lo que compra.

## Ampliación (no está en OSTEP)

- **ZFS** es el capítulo hecho sistema: checksums referenciales en todo el árbol (fletcher4/sha256), self-healing (al detectar corrupción en una copia, la repara desde el espejo automáticamente), scrub programable. Btrfs sigue el modelo (crc32c/xxhash); ext4/XFS solo protegen *metadatos* con checksums — tus datos, no. dm-integrity es la opción a nivel de bloque en Linux.
- End-to-end: la lección de Saltzer, Reed & Clark (*End-to-End Arguments in System Design*, 1984, citada por el capítulo como principio general) — la verificación vale de verdad cuando la hace la capa más alta posible; cualquier capa intermedia puede corromper después de verificar. Por eso las bases de datos serias (y los formatos como Parquet) añaden sus propios checksums encima del FS.
- La memoria también miente: los **bit flips de RAM** (rayos cósmicos, Rowhammer) corrompen antes de llegar al disco — ECC RAM es el complemento necesario del checksum de disco (los estudios de Google sobre DRAM: errores mucho más comunes de lo asumido).

## Referencias

- OSTEP cap. 45.
- Bairavasundaram et al., *An Analysis of Latent Sector Errors in Disk Drives*, SIGMETRICS 2007; *An Analysis of Data Corruption in the Storage Stack*, FAST 2008.
- Zhang et al., *End-to-end Data Integrity for File Systems: A ZFS Case Study*, FAST 2010.
- Saltzer, Reed & Clark, *End-to-End Arguments in System Design*, ACM TOCS 1984.
