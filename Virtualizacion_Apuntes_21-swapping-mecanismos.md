# 21 · Más allá de la memoria física: mecanismos (swap)

> Apuntes propios sobre el capítulo 21 de OSTEP (*Beyond Physical Memory: Mechanisms*). Fuente: https://ostep.org

## 21.1 Retirar el último supuesto

Hasta ahora asumíamos que todo espacio de direcciones cabe en RAM. Retirado el supuesto, el SO usa el **disco** como extensión: la jerarquía de memoria gana un nivel más (registros → cachés → RAM → disco). Objetivo: soportar espacios de direcciones **grandes** con la ilusión intacta — el programa no gestiona ningún "overlay" manual como antaño; el SO mueve páginas por debajo.

Nota clave: el swap no solo permite procesos gigantes; permite **más procesos a la vez** (multiprogramación agresiva) manteniendo en RAM solo lo que cada uno toca de verdad.

## 21.2 Espacio de swap y el bit presente

- **Swap space**: región de disco reservada para páginas expulsadas (partición o fichero de swap). El SO recuerda la dirección de disco de cada página expulsada.
- El mecanismo se cuelga del **bit presente** de la PTE (cap. 18): página en RAM → P=1, traducción normal; página expulsada → P=0 y la PTE guarda (en vez del PFN) dónde está en disco.

## 21.3 El fallo de página

Acceso a página con P=0 → excepción de **page fault** → corre el manejador del SO (siempre el SO, aunque la TLB sea hardware-managed):

1. Localiza la página en swap (con la info de la PTE).
2. Busca un marco libre; si no hay, **expulsa** otra página primero (la política: cap. 22).
3. Lanza la lectura de disco. El proceso queda **bloqueado** — el SO ejecuta otros mientras tanto (solapamiento, siempre).
4. Llega la página: actualiza la PTE (P=1, PFN nuevo), reintenta la instrucción que falló — ahora habrá miss de TLB, se rellena, y el acceso completa. El proceso no ha notado nada, salvo el tiempo.

Coste: un acceso a RAM ~100 ns; un fallo de página con disco ~ms → **decenas de miles de veces más lento**. La ilusión es transparente en semántica, no en rendimiento.

## 21.4 Gestión proactiva: marca de agua y el demonio de swap

El SO no espera a que la RAM esté 100% llena: mantiene un colchón con dos umbrales — **low watermark** y **high watermark**. Al caer los marcos libres por debajo del LW, un hilo del kernel (*swap daemon* / `kswapd` en Linux) despierta y expulsa páginas hasta recuperar el HW. Ventajas: las expulsiones se hacen **por lotes** (agrupar escrituras a disco amortiza), y fuera del camino crítico del fallo.

Detalle: expulsar una página **limpia** (no modificada desde que llegó, bit D=0) es gratis — ya hay copia idéntica en disco/fichero, basta descartarla; una **sucia** exige escribirla primero. El bit dirty existe para esto.

## Ampliación (no está en OSTEP)

- El fallo de página es la navaja suiza del SO moderno, no solo swap: **demand paging** de ejecutables (arranca el programa sin cargar nada; las páginas de código llegan al tocarlas, desde el fichero — el binario es su propio "swap"), **copy-on-write** de `fork()`, páginas de ceros perezosas para malloc, `mmap` de ficheros. Lo mismo, con distinto relleno. (Ver `../virtualizacion.md` §3.3.)
- Linux hoy: `vmstat 1` (columnas si/so = swap-in/out), `/proc/meminfo`, `swappiness` (0–200: cuánto prefiere el kernel expulsar páginas anónimas vs de fichero). Con SSD/NVMe el castigo del swap baja de ~10 ms a ~100 µs — sigue siendo ×1000 la RAM.
- Alternativas modernas al swap a disco: **zswap/zram** (comprimir páginas en RAM en vez de escribirlas: ×3–4 de "RAM extra" a coste de CPU; estándar en Android y ChromeOS), y en datacenter, swap a memoria remota vía RDMA (far memory, p. ej. el sistema de Google en *Software-defined far memory*, ASPLOS 2019).
- El **OOM killer** de Linux: cuando ni expulsando se libera suficiente, el kernel mata al proceso con peor puntuación (`/proc/<pid>/oom_score`) — la confesión de que el overcommit es una apuesta.

## Referencias

- OSTEP cap. 21.
- Kilburn et al., *One-level storage system* (Atlas 1962) — el primer demand paging.
- Denning, *Virtual Memory*, Computing Surveys 1970.
- `Documentation/admin-guide/mm/` del kernel (zswap, concepts).
