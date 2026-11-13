---
title: Virtualización y sistemas operativos
---

# Apuntes de OSTEP en español

Apuntes de estudio **originales, escritos con mis propias palabras**, que cubren el temario de *Operating Systems: Three Easy Pieces* (OSTEP), de Remzi y Andrea Arpaci-Dusseau. **No son una traducción del libro**: son notas propias que siguen su estructura de capítulos, amplían con material moderno (Linux actual, hardware actual) y remiten siempre al capítulo original.

El libro es gratuito en [ostep.org](https://ostep.org) (versión 1.10). Léelo como fuente primaria; usa estos apuntes como guía de estudio en español y como resumen ampliado.

## Convenciones

- Cada fichero indica a qué capítulo de OSTEP corresponde.
- Los capítulos-diálogo del libro (1, 3, 11, 12, 24, 25, 34, 35, 46, 47, 51) no tienen contenido técnico propio; su idea se integra en el capítulo siguiente y no tienen fichero aparte.
- Sección "Ampliación" en cada fichero: material que no está en OSTEP (o está desactualizado allí), siempre con referencias.

## Índice de módulos

### Parte 0 — Introducción
| Módulo | Capítulo OSTEP |
|---|---|
| [02 — Introducción](Virtualizacion_Apuntes_02-introduccion) | 2. Introduction to Operating Systems |

### Parte I — Virtualización · CPU
| Módulo | Capítulo OSTEP |
|---|---|
| [04 — El proceso](Virtualizacion_Apuntes_04-procesos) | 4. The Abstraction: The Process |
| [05 — API de procesos](Virtualizacion_Apuntes_05-api-de-procesos) | 5. Interlude: Process API |
| [06 — Ejecución directa limitada](Virtualizacion_Apuntes_06-ejecucion-directa-limitada) | 6. Mechanism: Limited Direct Execution |
| [07 — Planificación: introducción](Virtualizacion_Apuntes_07-planificacion-introduccion) | 7. Scheduling: Introduction |
| [08 — MLFQ](Virtualizacion_Apuntes_08-mlfq) | 8. Scheduling: The Multi-Level Feedback Queue |
| [09 — Planificación proporcional](Virtualizacion_Apuntes_09-planificacion-proporcional) | 9. Scheduling: Proportional Share |
| [10 — Planificación multiprocesador](Virtualizacion_Apuntes_10-planificacion-multiprocesador) | 10. Multiprocessor Scheduling |

### Parte I — Virtualización · Memoria
| Módulo | Capítulo OSTEP |
|---|---|
| [13 — Espacios de direcciones](Virtualizacion_Apuntes_13-espacios-de-direcciones) | 13. The Abstraction: Address Spaces |
| [14 — API de memoria](Virtualizacion_Apuntes_14-api-de-memoria) | 14. Interlude: Memory API |
| [15 — Traducción de direcciones](Virtualizacion_Apuntes_15-traduccion-de-direcciones) | 15. Mechanism: Address Translation |
| [16 — Segmentación](Virtualizacion_Apuntes_16-segmentacion) | 16. Segmentation |
| [17 — Gestión del espacio libre](Virtualizacion_Apuntes_17-gestion-de-espacio-libre) | 17. Free-Space Management |
| [18 — Paginación: introducción](Virtualizacion_Apuntes_18-paginacion) | 18. Paging: Introduction |
| [19 — TLB](Virtualizacion_Apuntes_19-tlb) | 19. Paging: Faster Translations (TLBs) |
| [20 — Tablas de páginas avanzadas](Virtualizacion_Apuntes_20-tablas-de-paginas-avanzadas) | 20. Paging: Smaller Tables |
| [21 — Swapping: mecanismos](Virtualizacion_Apuntes_21-swapping-mecanismos) | 21. Beyond Physical Memory: Mechanisms |
| [22 — Swapping: políticas](Virtualizacion_Apuntes_22-swapping-politicas) | 22. Beyond Physical Memory: Policies |
| [23 — Sistemas de VM completos](Virtualizacion_Apuntes_23-sistemas-vm-completos) | 23. Complete Virtual Memory Systems |

### Parte II — Concurrencia
| Módulo | Capítulo OSTEP |
|---|---|
| [26 — Concurrencia: introducción](Virtualizacion_Apuntes_26-concurrencia-introduccion) | 26. Concurrency: An Introduction |
| [27 — API de hilos](Virtualizacion_Apuntes_27-api-de-hilos) | 27. Interlude: Thread API |
| [28 — Locks](Virtualizacion_Apuntes_28-locks) | 28. Locks |
| [29 — Estructuras con locks](Virtualizacion_Apuntes_29-estructuras-con-locks) | 29. Lock-based Concurrent Data Structures |
| [30 — Variables de condición](Virtualizacion_Apuntes_30-variables-de-condicion) | 30. Condition Variables |
| [31 — Semáforos](Virtualizacion_Apuntes_31-semaforos) | 31. Semaphores |
| [32 — Bugs de concurrencia](Virtualizacion_Apuntes_32-bugs-de-concurrencia) | 32. Common Concurrency Problems |
| [33 — Concurrencia por eventos](Virtualizacion_Apuntes_33-concurrencia-por-eventos) | 33. Event-based Concurrency |

### Parte III — Persistencia
| Módulo | Capítulo OSTEP |
|---|---|
| [36 — Dispositivos de E/S](Virtualizacion_Apuntes_36-dispositivos-es) | 36. I/O Devices |
| [37 — Discos duros](Virtualizacion_Apuntes_37-discos-duros) | 37. Hard Disk Drives |
| [38 — RAID](Virtualizacion_Apuntes_38-raid) | 38. RAID |
| [39 — Ficheros y directorios](Virtualizacion_Apuntes_39-ficheros-y-directorios) | 39. Interlude: Files and Directories |
| [40 — Implementación de FS](Virtualizacion_Apuntes_40-implementacion-de-fs) | 40. File System Implementation |
| [41 — FFS](Virtualizacion_Apuntes_41-ffs) | 41. Locality and The Fast File System |
| [42 — Consistencia tras fallos](Virtualizacion_Apuntes_42-consistencia-tras-fallos) | 42. Crash Consistency: FSCK and Journaling |
| [43 — LFS](Virtualizacion_Apuntes_43-lfs) | 43. Log-structured File Systems |
| [44 — SSD](Virtualizacion_Apuntes_44-ssd) | 44. Flash-based SSDs |
| [45 — Integridad de datos](Virtualizacion_Apuntes_45-integridad-de-datos) | 45. Data Integrity and Protection |

### Parte IV — Distribución
| Módulo | Capítulo OSTEP |
|---|---|
| [48 — Sistemas distribuidos](Virtualizacion_Apuntes_48-sistemas-distribuidos) | 48. Distributed Systems |
| [49 — NFS](Virtualizacion_Apuntes_49-nfs) | 49. Sun's Network File System (NFS) |
| [50 — AFS](Virtualizacion_Apuntes_50-afs) | 50. The Andrew File System (AFS) |

### Apéndices
| Módulo | Apéndice OSTEP |
|---|---|
| [Apéndice B — Monitores de máquinas virtuales](Virtualizacion_Apuntes_ap-vmm) | B. Virtual Machine Monitors |

## Referencia principal

Arpaci-Dusseau, R. & A., *Operating Systems: Three Easy Pieces*, v1.10, Arpaci-Dusseau Books. Gratis en [ostep.org](https://ostep.org). Cada capítulo del libro incluye además su propia bibliografía y **homeworks** con simuladores ([ostep-homework](https://github.com/remzi-arpacidusseau/ostep-homework)) — hacerlos vale más que releer.
