# Virtualización y sistemas operativos

[![Página web](https://img.shields.io/badge/Página_web-ver-4A90E2?style=flat-square&logo=github)](https://amadorprof.github.io/Virtualizacion/)
[![Licencia CC BY 4.0](https://img.shields.io/badge/Licencia-CC%20BY%204.0-green?style=flat-square)](https://creativecommons.org/licenses/by/4.0/)

Apuntes de estudio originales, escritos con mis propias palabras, que cubren el temario de *Operating Systems: Three Easy Pieces* (OSTEP), de Remzi y Andrea Arpaci-Dusseau. No son una traducción del libro: son notas propias que siguen su estructura de capítulos, amplían con material moderno y remiten siempre al capítulo original.

El libro es gratuito en [ostep.org](https://ostep.org) (versión 1.10).

## Índice de módulos

| Módulo | Capítulo OSTEP | Bloque |
|---|---|---|
| `..._02-introduccion.md` | 2. Introduction to Operating Systems | Introducción |
| `..._04-procesos.md` | 4. The Abstraction: The Process | CPU |
| `..._05-api-de-procesos.md` | 5. Interlude: Process API | CPU |
| `..._06-ejecucion-directa-limitada.md` | 6. Mechanism: Limited Direct Execution | CPU |
| `..._07-planificacion-introduccion.md` | 7. Scheduling: Introduction | CPU |
| `..._08-mlfq.md` | 8. The Multi-Level Feedback Queue | CPU |
| `..._09-planificacion-proporcional.md` | 9. Scheduling: Proportional Share | CPU |
| `..._10-planificacion-multiprocesador.md` | 10. Multiprocessor Scheduling | CPU |
| `..._13-espacios-de-direcciones.md` | 13. The Abstraction: Address Spaces | Memoria |
| `..._14-api-de-memoria.md` | 14. Interlude: Memory API | Memoria |
| `..._15-traduccion-de-direcciones.md` | 15. Mechanism: Address Translation | Memoria |
| `..._16-segmentacion.md` | 16. Segmentation | Memoria |
| `..._17-gestion-de-espacio-libre.md` | 17. Free-Space Management | Memoria |
| `..._18-paginacion.md` | 18. Paging: Introduction | Memoria |
| `..._19-tlb.md` | 19. Paging: Faster Translations (TLBs) | Memoria |
| `..._20-tablas-de-paginas-avanzadas.md` | 20. Paging: Smaller Tables | Memoria |
| `..._21-swapping-mecanismos.md` | 21. Beyond Physical Memory: Mechanisms | Memoria |
| `..._22-swapping-politicas.md` | 22. Beyond Physical Memory: Policies | Memoria |
| `..._23-sistemas-vm-completos.md` | 23. Complete Virtual Memory Systems | Memoria |
| `..._26-concurrencia-introduccion.md` | 26. Concurrency: An Introduction | Concurrencia |
| `..._27-api-de-hilos.md` | 27. Interlude: Thread API | Concurrencia |
| `..._28-locks.md` | 28. Locks | Concurrencia |
| `..._29-estructuras-con-locks.md` | 29. Lock-based Concurrent Data Structures | Concurrencia |
| `..._30-variables-de-condicion.md` | 30. Condition Variables | Concurrencia |
| `..._31-semaforos.md` | 31. Semaphores | Concurrencia |
| `..._32-bugs-de-concurrencia.md` | 32. Common Concurrency Problems | Concurrencia |
| `..._33-concurrencia-por-eventos.md` | 33. Event-based Concurrency | Concurrencia |
| `..._36-dispositivos-es.md` | 36. I/O Devices | Almacenamiento |
| `..._37-discos-duros.md` | 37. Hard Disk Drives | Almacenamiento |
| `..._38-raid.md` | 38. RAID | Almacenamiento |
| `..._39-ficheros-y-directorios.md` | 39. Interlude: Files and Directories | Almacenamiento |
| `..._40-implementacion-de-fs.md` | 40. File System Implementation | Almacenamiento |
| `..._41-ffs.md` | 41. Locality and The Fast File System | Almacenamiento |
| `..._42-consistencia-tras-fallos.md` | 42. Crash Consistency: FSCK and Journaling | Almacenamiento |
| `..._43-lfs.md` | 43. Log-structured File Systems | Almacenamiento |
| `..._44-ssd.md` | 44. Flash-based SSDs | Almacenamiento |
| `..._45-integridad-de-datos.md` | 45. Data Integrity and Protection | Almacenamiento |
| `..._48-sistemas-distribuidos.md` | 48. Distributed Systems | Distribución |
| `..._49-nfs.md` | 49. Sun's Network File System (NFS) | Distribución |
| `..._50-afs.md` | 50. The Andrew File System (AFS) | Distribución |
| `..._ap-vmm.md` | B. Virtual Machine Monitors | Apéndice |

## Convenciones

Cada fichero indica a qué capítulo de OSTEP corresponde. Los capítulos-diálogo del libro (1, 3, 11, 12, 24, 25, 34, 35, 46, 47, 51) no tienen contenido técnico propio y se integran en el capítulo siguiente, por lo que no tienen fichero propio. La sección "Ampliación" de cada módulo recoge material que no está en OSTEP o está desactualizado allí, siempre con referencias.

---

Publicado bajo [Creative Commons Attribution 4.0 International](https://creativecommons.org/licenses/by/4.0/) — puedes usar, compartir y adaptar con atribución.
