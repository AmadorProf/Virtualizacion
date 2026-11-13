# 23 · Sistemas de memoria virtual completos: VAX/VMS y Linux

> Apuntes propios sobre el capítulo 23 de OSTEP (*Complete Virtual Memory Systems*). Fuente: https://ostep.org

Este capítulo junta todas las piezas (traducción, TLB, tablas, swap, políticas) viendo dos sistemas reales: uno histórico ejemplar (VAX/VMS, años 70-80) y uno actual (Linux x86-64). La gracia está en ver cuánta ingeniería "extra" exige un sistema completo frente a los esquemas de pizarra.

## 23.1 VAX/VMS: elegancia sobre hardware imperfecto

Contexto: el VAX-11 debía correr VMS desde minis baratos a mainframes; el software debía compensar las carencias del hardware. Lecciones que dio:

- **Espacio de direcciones híbrido**: proceso partido en P0 (código+heap) y P1 (pila), con la mitad alta para el **sistema** — el kernel mapeado *dentro* del espacio de cada proceso (protegido por bits de privilegio). Esto facilita que el kernel acceda a datos de usuario y persiste hoy en casi todos los SO.
- **Tablas de páginas en memoria virtual del kernel**: las tablas de usuario viven en espacio paginable del sistema → pueden swapearse; menos presión de RAM a costa de walks más complejos.
- **Página 0 inválida**: el acceso a NULL falla porque la primera página se marca inválida a propósito — el origen del segfault por puntero nulo, pura convención de SO.
- **Reemplazo sin bit de referencia**: el VAX no tenía bit A. VMS inventó el **FIFO segmentado por proceso**: cada proceso tiene un máximo de marcos (RSS) gestionados FIFO, y las páginas expulsadas van a dos **listas globales de segunda oportunidad** (limpias/sucias) desde donde pueden rescatarse sin ir a disco — un "LRU global aproximado" hecho por software. Bonus: la lista de sucias permite **clustering** de escrituras.
- **Demand zeroing y COW**: la página nueva se pone a cero *solo si se toca*; la compartición copia *solo si se escribe*. La pereza (*laziness*) como principio de diseño — hacer el trabajo solo cuando es inevitable, y a veces nunca.

## 23.2 La memoria virtual de Linux (x86-64)

- **Layout**: mitad baja del espacio canónico para usuario, mitad alta para kernel — VMS otra vez. Dentro del kernel: la **logical memory** (mapeo directo 1:1 de la RAM, para estructuras del kernel y DMA) vs **vmalloc** (virtualmente contigua, físicamente dispersa).
- **Tablas de 4/5 niveles**, huge pages (THP y explícitas) por la cobertura de TLB (caps. 19-20).
- **Page cache unificada**: las páginas de fichero (mmap y read/write), las anónimas (heap/pila) y las del propio kernel comparten un mismo sistema de caching con listas activa/inactiva (2Q-like; hoy MGLRU) y writeback de sucias por umbral.
- **Seguridad, la gran adición moderna**: buena parte del diseño actual no la dictó el rendimiento sino los ataques —
  - **Buffer overflows** → NX (pilas/heap no ejecutables);
  - *return-oriented programming* (ROP: encadenar trocitos de código existente) → **ASLR** (aleatorizar pila/heap/librerías);
  - **Meltdown/Spectre** (2018) → **KPTI**: separar (parcialmente) las tablas del kernel de las de usuario — deshaciendo la optimización de VMS de mapear el kernel en cada proceso. La historia de la MV moderna es en parte una carrera armamentística.

## 23.3 Moralejas del capítulo

1. Un sistema real es la *suma* de decenas de mecanismos pequeños (lazy zeroing, clustering, listas de rescate) — no una política de pizarra.
2. Las buenas ideas se **heredan** entre sistemas separados por 40 años (VMS → Linux: layout, COW, segunda oportunidad).
3. El adagio de Lampson que cita el libro: las ideas se **reusan**; entender los clásicos es leer el presente con ventaja.
4. La seguridad ya es un requisito de primer orden del subsistema de memoria, no un añadido.

## Ampliación (no está en OSTEP)

- El capítulo cierra el círculo con la virtualización anidada: EPT/NPT son "otro VMS más abajo" — cada mecanismo de este capítulo (COW, demand paging, listas) existe duplicado en el hipervisor sobre la "RAM física" del guest (ver `../virtualizacion.md` §5.5 y §6.4).
- Frontera actual de Linux MM: **folios** (gestionar grupos de páginas como unidad, 5.16+), **DAMON** (monitorización de acceso para políticas), **CXL memory tiering** (RAM lejana conectada por bus — el "swap a memoria" hecho hardware).
- Para jugar: `pmap -x <pid>`, `smem`, `/proc/<pid>/smaps_rollup` (RSS/PSS/USS: cuánta memoria es "de verdad" de un proceso con tanta compartición — pregunta sin respuesta única, herencia directa del COW).

## Referencias

- OSTEP cap. 23.
- Levy & Lipman, *Virtual Memory Management in the VAX/VMS Operating System*, IEEE Computer 1982 — el paper que estructura la primera mitad.
- Gorman, *Understanding the Linux Virtual Memory Manager* (libre online) — anticuado pero aún la mejor visión de conjunto.
- Lipp et al., *Meltdown* y Kocher et al., *Spectre Attacks*, S&P/USENIX 2018-19.
- `Documentation/mm/` del kernel Linux.
