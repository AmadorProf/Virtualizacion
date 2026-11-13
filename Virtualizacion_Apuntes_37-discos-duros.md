# 37 · Discos duros

> Apuntes propios sobre el capítulo 37 de OSTEP (*Hard Disk Drives*). Fuente: https://ostep.org

## 37.1 La interfaz

Un disco expone un array de **sectores** (512 B) numerados 0..N−1, con escritura atómica por sector (una escritura multi-sector puede quedar a medias si se va la luz — semilla del cap. 42). Contrato implícito: los sectores cercanos en número están cerca físicamente, y el acceso secuencial es mucho más rápido que el aleatorio.

## 37.2 La geometría

- **Platos** con dos superficies magnéticas, girando solidarios a velocidad fija (5400–15000 **RPM**).
- Cada superficie: **pistas** concéntricas; cada pista: sectores. Un **cabezal** por superficie, todos en un mismo **brazo** que se desplaza radialmente.

El tiempo de una petición = tres componentes:

1. **Seek** (mover el brazo a la pista): ~4–10 ms medio — acelerar, desplazar, frenar, asentar (el *settling* solo ya es ~0.5–2 ms).
2. **Rotación** (esperar a que el sector pase bajo el cabezal): media = media vuelta; a 7200 RPM una vuelta son 8.3 ms → ~4.2 ms medios.
3. **Transferencia**: leer/escribir al vuelo — ~decenas de µs. Despreciable frente a lo mecánico.

## 37.3 Consecuencia central: secuencial ≫ aleatorio

Con seek+rotación ~10 ms por petición aleatoria, un disco hace ~100 IOPS aleatorias; en secuencial, mueve 100–200 MB/s. El cálculo del capítulo con discos reales: el disco de alto rendimiento rinde ~0.7 MB/s en aleatorio vs ~125 MB/s en secuencial — **dos órdenes de magnitud**. Toda la ingeniería de sistemas de ficheros de los caps. 41–43 existe para convertir cargas aleatorias en secuenciales.

Detalles de diseño que completan el cuadro: **track skew** (desfasar el sector 0 de pistas contiguas para no perder una vuelta al cambiar de pista en lecturas secuenciales), **zonas multi-densidad** (las pistas exteriores, más largas, llevan más sectores → más MB/s fuera que dentro), y la **caché del disco** (track buffer, MiBs) — con la cuestión espinosa del *write-back* (reportar "escrito" con el dato aún en caché volátil: rápido y peligroso; base de los problemas del cap. 42).

## 37.4 Scheduling de disco

Con varias peticiones pendientes, el orden importa muchísimo (a diferencia del scheduling de CPU, aquí la duración de cada "trabajo" es estimable por posición → cuasi-SJF):

- **SSTF** (shortest seek first): la petición de pista más cercana. Bueno, pero **inanición** de las lejanas.
- **Elevator/SCAN** (y C-SCAN): barrer el disco de fuera a dentro y volver, atendiendo lo que pille — el algoritmo del ascensor; evita la inanición. Pero ignora la rotación.
- **SPTF** (shortest positioning time first): decide con seek+rotación reales. Óptimo local; requiere conocer la geometría → se hace **dentro del disco** (el SO ya no ve la geometría real; envía un lote y el firmware ordena).
- Trabajo del SO moderno: **fusionar** peticiones contiguas y decidir cuánto esperar acumulando lote (*anticipatory scheduling*: a veces esperar un poco gana).

## Ampliación (no está en OSTEP)

- Los discos siguen vivos en 2026 como capa de capacidad (~$/TB imbatible): 20–30 TB con grabación **SMR** (pistas solapadas: escritura aleatoria aún peor, zonas append-only — el disco se vuelve "log-structured", cap. 43 profético) y **HAMR** (calentamiento láser para densidades mayores).
- Schedulers reales de Linux hoy: mq-deadline, BFQ, Kyber (multi-cola, pensados también para NVMe); para SSDs rápidos, a menudo `none` — el scheduling de posición ya no aplica sin mecánica.
- Regla de bolsillo vigente: HDD ~100–200 IOPS aleatorias; SSD SATA ~50–100k; NVMe ~500k–1M+. Cinco órdenes de magnitud que explican una década de rediseños de software (cap. 44).

## Referencias

- OSTEP cap. 37.
- Ruemmler & Wilkes, *An Introduction to Disk Drive Modeling*, IEEE Computer 1994 — el clásico de modelado.
- Worthington, Ganger & Patt, *Scheduling Algorithms for Modern Disk Drives*, SIGMETRICS 1994.
- `Documentation/block/` del kernel Linux.
