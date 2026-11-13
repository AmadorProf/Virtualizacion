# 36 · Dispositivos de E/S

> Apuntes propios sobre el capítulo 36 de OSTEP (*I/O Devices*). Fuente: https://ostep.org

## 36.1 Arquitectura del sistema

La CPU no habla con los dispositivos directamente sino a través de **buses jerárquicos**: cuanto más rápido el bus, más corto y caro → lo rápido cerca de la CPU (memoria), lo lento lejos (periféricos). PC moderno: bus de memoria → PCIe para lo veloz (GPU, NVMe) → SATA/USB para el resto. La jerarquía es física: la velocidad de la luz y el coste de las pistas mandan.

## 36.2 El dispositivo canónico

Todo dispositivo expone una **interfaz** (registros visibles) y esconde una **implementación** (su hardware + firmware; los complejos llevan su propio microcontrolador). Interfaz mínima de tres registros:

- **status**: ¿ocupado? ¿error?
- **command**: qué hacer.
- **data**: los datos a mover.

Protocolo ingenuo del SO (los 4 pasos): esperar a no-ocupado (polling) → escribir datos → escribir comando → esperar a que acabe.

## 36.3 Tres optimizaciones estructurales

**a) Interrupciones en vez de polling.** El polling quema CPU esperando. Con **interrupciones**, el SO duerme al proceso y sigue con otro; el dispositivo interrumpe al terminar → solapamiento de CPU y E/S. Matices importantes:

- Para dispositivos **rápidos**, la interrupción puede ser *peor* (el cambio de contexto cuesta más que la espera): polling gana. Híbrido habitual: girar un poco y dormir después.
- **Livelock de interrupciones** (red a alta carga): la CPU se pasa la vida atendiendo interrupciones sin procesar nada. Solución: conmutar a polling bajo carga (**NAPI** en Linux hace exactamente esto).
- **Coalescing**: el dispositivo agrupa varias completaciones en una sola interrupción — throughput a cambio de latencia.

**b) DMA.** Copiar datos con la CPU (PIO) desperdicia sus ciclos. El motor de **DMA** copia RAM↔dispositivo por su cuenta; la CPU solo programa la transferencia y recibe la interrupción final.

**c) Acceso a los registros**: instrucciones de E/S dedicadas (`in`/`out` x86, privilegiadas) o **MMIO** (los registros mapeados como direcciones de memoria; hoy dominante).

## 36.4 Drivers: encajar mil dispositivos en un SO

El **driver** encapsula los detalles del dispositivo bajo interfaces neutras del kernel (la pila de almacenamiento: FS → bloque genérico → driver). Dos observaciones del capítulo con moraleja:

- La abstracción tiene coste: una interfaz genérica de bloques no puede explotar particularidades ricas de un dispositivo (SCSI y sus errores detallados reducidos a un errno).
- Los drivers son ~70% del código del kernel Linux, muchos escritos por aficionados → **la mayor fuente de bugs del kernel** (estudios empíricos citados: Chou et al., SOSP 2001).

## 36.5 Ejemplo: IDE

El capítulo cierra con el driver IDE real de xv6 (~150 líneas): colas de peticiones, escribir registros, esperar interrupción — los 4 pasos del protocolo hechos código. Vale la pena leerlo en el repo de xv6: es un driver completo abarcable.

## Ampliación (no está en OSTEP)

- NVMe es el rediseño de la interfaz para SSDs: **colas de envío/completado en RAM compartida** (hasta 64k colas × 64k comandos, una cola por núcleo, sin locks) + doorbells MMIO — el mismo patrón de anillos de virtio e io_uring. La interfaz importa tanto como el medio: SATA limitaba los SSD tanto como su flash.
- MSI/MSI-X: las interrupciones modernas son *mensajes* PCIe (sin pines), dirigibles a la CPU que interesa — clave para repartir la carga de red entre núcleos (RSS).
- En virtualización, todo este capítulo es lo que el hipervisor debe falsificar — y por qué existen virtio (paravirtualizar los 3 registros y el DMA) y el passthrough con IOMMU (`../virtualizacion.md` §5.6).

## Referencias

- OSTEP cap. 36.
- Chou et al., *An Empirical Study of Operating System Errors*, SOSP 2001.
- Corbet, Rubini & Kroah-Hartman, *Linux Device Drivers* (3ª ed., libre online).
- Especificación NVMe (nvmexpress.org) para el contraste moderno.
