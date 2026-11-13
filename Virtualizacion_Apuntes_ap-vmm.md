# Apéndice B · Monitores de máquinas virtuales (VMM)

> Apuntes propios sobre el apéndice de OSTEP dedicado a los VMM (*Virtual Machine Monitors*). Fuente: https://ostep.org. Complemento directo de `../virtualizacion.md` (§5–6), que cubre el tema con más detalle moderno; aquí, la perspectiva OSTEP: el VMM como "el SO de los SO".

## B.1 La idea: virtualizar una capa más abajo

Todo el libro trata de cómo el SO virtualiza CPU y memoria para los procesos. El VMM (hipervisor) aplica **exactamente la misma jugada un nivel más abajo**: presenta a cada *sistema operativo invitado* la ilusión de una máquina física propia, multiplexando la real entre varios SO.

Motivaciones (las de IBM en los 60 y las del renacimiento ~2000): consolidación de servidores (muchos SO infrautilizados en una máquina), compatibilidad (correr apps de otro SO), aislamiento fuerte, y facilidad de gestión/migración.

Requisito rector (la transparencia del cap. 13, elevada): el guest **no debe notar** que está virtualizado, ni el VMM exigir modificarlo.

## B.2 Virtualizar la CPU: trap-and-emulate de dos pisos

El guest debe correr **sin privilegio real** (si tocara la máquina de verdad, adiós ilusión). El mecanismo: ejecución directa limitada… con un piso más:

- Una aplicación del guest hace una **syscall** → el trap NO va al kernel del guest (no controla la tabla de traps real) sino al **VMM**, que salta al manejador que el guest *cree* haber instalado (el VMM lo apuntó cuando el guest "instaló su tabla de traps" — esa instalación fue, a su vez, una operación privilegiada atrapada).
- El kernel del guest, al ejecutar sus instrucciones privilegiadas (return-from-trap, cargar tablas de páginas…), vuelve a atrapar en el VMM, que **emula** el efecto sobre el estado virtual.

Problema histórico que el apéndice señala: en x86 clásico algunas instrucciones sensibles **no atrapaban** al ejecutarse sin privilegio (fallaban en silencio) → trap-and-emulate puro imposible; VMware lo sorteó con traducción binaria y, desde 2005, VT-x/AMD-V lo arreglan en hardware (detalle completo en `../virtualizacion.md` §5.2–5.4). Truco de rendimiento que menciona: usar niveles de privilegio intermedios (ring 1) o sombras de estado para no pagar dos traps por cada syscall del guest.

## B.3 Virtualizar la memoria: tres niveles de direcciones

El guest ya traducía virtual→"física"; ahora la "física" del guest es otra ilusión → tercera capa: **física del guest → física real de la máquina**. El VMM mantiene por cada VM la traducción extra, clásicamente con **shadow page tables**: el VMM intercepta las modificaciones que el guest hace a sus tablas y mantiene en secreto tablas combinadas (virtual del guest → física real) que son las que carga en la MMU de verdad.

- El bit presente cobra doble sentido: una página puede faltar "para el guest" (su swap) o "para el VMM" (el VMM la expulsó sin que el guest lo sepa) — dos gestores de memoria apilados que no se conocen.
- La versión moderna por hardware (EPT/NPT) y sus costes están en `../virtualizacion.md` §5.5.
- Perla histórica que cuenta el apéndice: VMM sobre un SO sin cooperación puede chocar con suposiciones del guest — el caso del information leak entre el VMM de Disco y el kernel de IRIX que asumía posiciones fijas de memoria; moraleja: la transparencia perfecta es escurridiza.

## B.4 La paravirtualización como atajo

Si se acepta modificar el guest ("para-virtualización"), el guest colabora con el VMM (hypercalls) y todo se abarata — el enfoque de Xen y de los drivers virtio actuales (`../virtualizacion.md` §5.3, §5.6). El apéndice lo menciona como el contrapunto: transparencia total vs rendimiento.

## B.5 La conexión con el resto del libro (y de tus apuntes)

| Concepto del libro | Su versión VMM |
|---|---|
| Ejecución directa limitada (cap. 6) | Directa limitada de dos pisos (VM entry/exit) |
| Context switch (cap. 6) | Cambio de VM / migración en vivo |
| Tablas de páginas (caps. 18–20) | Shadow pages / EPT |
| Swap (cap. 21) | Ballooning, doble paginación |
| Scheduler (caps. 7–10) | Scheduler de vCPUs |

El VMM demuestra la tesis silenciosa de todo OSTEP: los mecanismos de virtualización son **recursivos** — quien sabe virtualizar procesos sabe virtualizar máquinas.

## Ampliación (no está en OSTEP)

- Todo el desarrollo moderno (VT-x/VMCS, EPT, virtio, KVM/QEMU, tipos 1 y 2, microVMs/Firecracker, nested) está en `../virtualizacion.md` §5–7 con referencias propias.
- Para el puente práctico: el ejercicio del mini-VMM con `/dev/kvm` de la ruta de aprendizaje (`../ruta-aprendizaje-virtualizacion.md`, Fase 3).

## Referencias

- OSTEP, apéndice *Virtual Machine Monitors*.
- Bugnion et al., *Disco: Running Commodity Operating Systems on Scalable Multiprocessors*, SOSP 1997 — el paper que revivió los VMM (y germen de VMware).
- Adams & Agesen, *A Comparison of Software and Hardware Techniques for x86 Virtualization*, ASPLOS 2006.
- Barham et al., *Xen and the Art of Virtualization*, SOSP 2003.
- Bugnion, Nieh & Tsafrir, *Hardware and Software Support for Virtualization*, 2017.
