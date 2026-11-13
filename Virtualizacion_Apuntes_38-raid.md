# 38 · RAID

> Apuntes propios sobre el capítulo 38 de OSTEP (*RAID*). Fuente: https://ostep.org

## 38.1 La idea

**RAID** (Redundant Array of Inexpensive Disks; Patterson, Gibson & Katz, 1988): combinar varios discos en un dispositivo lógico que parece **un disco grande** pero es más rápido, más grande y/o más fiable. Transparencia total: el FS de arriba no cambia — una de las claves de su éxito comercial.

Internamente un RAID es casi un ordenador (controladora, RAM, a veces NVRAM) que traduce E/S lógicas a físicas — otra capa de virtualización: direcciones de bloque lógicas → (disco, offset).

Tres ejes de evaluación: **capacidad** útil, **fiabilidad** (cuántos fallos tolera) y **rendimiento** (secuencial vs aleatorio, lectura vs escritura). Modelo de fallo del capítulo: **fail-stop** — un disco funciona o está claramente muerto (los fallos sutiles, corrupción silenciosa, quedan para el cap. 45).

## 38.2 Los niveles clásicos

**RAID-0 (striping).** Los bloques se reparten en rotación entre N discos (con *chunk size* configurable). Sin redundancia: capacidad N, rendimiento ~N× en todo… y **fiabilidad peor que un disco** (cualquier fallo pierde datos). La vara de medir del rendimiento.

**RAID-1 (mirroring).** Cada bloque en 2 discos. Capacidad N/2; tolera 1 fallo por pareja; lecturas ~N× (cualquier copia sirve, reparto de carga), escrituras N/2× (hay que escribir ambas copias — en paralelo, pero la latencia es la peor de las dos).

**RAID-4 (paridad dedicada).** N−1 discos de datos + 1 de **paridad** (XOR de la franja): con un disco perdido, se reconstruye XOR-eando el resto. Capacidad (N−1); barato en espacio. Problema letal: la **escritura pequeña** — actualizar un bloque exige leer bloque viejo + paridad vieja, calcular (método sustractivo: P_nuevo = P_viejo ⊕ D_viejo ⊕ D_nuevo) y escribir ambos → **el disco de paridad participa en TODAS las escrituras** → cuello de botella absoluto (rendimiento de escritura aleatoria = el de un solo disco / 2).

**RAID-5 (paridad rotada).** La paridad rota entre todos los discos → el cuello de botella se difumina; escrituras aleatorias escalan ~N/4 (cada escritura pequeña sigue costando 4 operaciones, pero repartidas). El nivel de paridad razonable; RAID-4 queda obsoleto salvo diseños que lo explotan (NetApp WAFL, que escribe siempre franjas completas).

## 38.3 Resumen comparativo

| | Capacidad | Tolera | Lect. aleatoria | Escr. aleatoria |
|---|---|---|---|---|
| RAID-0 | N | 0 | ~N× | ~N× |
| RAID-1 | N/2 | ≥1 | ~N× | ~N/2× |
| RAID-5 | N−1 | 1 | ~N× | ~N/4× (4 ops/escritura) |

Regla práctica del capítulo: rendimiento y capacidad → striping; escritura aleatoria intensa + fiabilidad → mirroring; capacidad + fiabilidad con escritura moderada → paridad.

## 38.4 Problemas transversales

- **Consistent-update problem**: una escritura que toca 2+ discos (dato+espejo, dato+paridad) puede quedar a medias si falla la luz → "write hole". Solución seria: NVRAM/journal en la controladora — anticipo del cap. 42.
- **Reconstrucción**: con un disco muerto, reconstruir estresa a los supervivientes durante horas/días; la ventana de segundo fallo es real con discos grandes → RAID-6 (doble paridad).

## Ampliación (no está en OSTEP)

- Hoy: **RAID-6** (dos paridades, Reed-Solomon; estándar con discos ≥8 TB por la ventana de reconstrucción), **RAID-10** (espejo+stripe: el preferido para bases de datos), y en escala cloud, **erasure coding** (k+m genérico: S3, Ceph, HDFS) — la generalización matemática de la paridad, con menos overhead que replicar ×3.
- Implementaciones: md/mdadm y LVM en Linux (software RAID: la controladora "hardware" ya no aporta apenas), y la síntesis moderna FS+RAID integrados: **ZFS** (RAID-Z evita el write hole escribiendo franjas de tamaño variable, copy-on-write) y Btrfs.
- El "write hole" de RAID-5 software sigue siendo real: md lo mitiga con journal/bitmap (`--write-journal`); ZFS lo elimina por diseño.

## Referencias

- OSTEP cap. 38.
- Patterson, Gibson & Katz, *A Case for Redundant Arrays of Inexpensive Disks (RAID)*, SIGMOD 1988.
- Chen et al., *RAID: High-Performance, Reliable Secondary Storage*, ACM Computing Surveys 1994 — el survey.
- Bonwick, *RAID-Z* (blog de Sun, 2005).
