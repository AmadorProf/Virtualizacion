# 9 · Planificación proporcional (lotería, stride, CFS)

> Apuntes propios sobre el capítulo 9 de OSTEP (*Scheduling: Proportional Share*). Fuente: https://ostep.org

## 9.1 Cambio de objetivo

MLFQ optimiza turnaround/respuesta. Los schedulers **proporcionales** persiguen otra cosa: que cada proceso reciba un **porcentaje garantizado** de CPU. Es la métrica natural cuando la CPU se *vende* o se reparte entre usuarios/VMs/contenedores: "A tiene el 50%, B el 30%, C el 20%".

La moneda común es el **ticket**: la fracción de tickets que posees = la fracción de CPU que te toca.

## 9.2 Lotería (lottery scheduling)

Idea de Waldspurger (1994): cada rodaja, el scheduler **sortea** un número entre los tickets totales; el proceso que posee el ticket ganador corre. Con A=75 tickets y B=25, a la larga A recibe ~75%.

- **Aleatorio, no determinista**: la proporción se cumple *en esperanza*; a corto plazo hay varianza (métrica de *unfairness* del capítulo: cuánto se desvía). Cuanto más largos los trabajos, más se acerca al reparto ideal.
- Ventajas del azar: sin estado por proceso (no hay historial que mantener), decisiones O(nº procesos) triviales, resistente a casos patológicos.
- Mecanismos elegantes asociados: **ticket currency** (cada usuario reparte "su moneda" entre sus procesos y el sistema convierte), **ticket transfer** (un cliente cede tickets al servidor que trabaja para él — remedio a la inversión de prioridad), **ticket inflation** (entre procesos que confían entre sí).
- Problema abierto: **asignación** — ¿quién decide cuántos tickets merece cada quien? El capítulo lo deja abierto; en la práctica lo decide el administrador (pesos, `nice`, límites de cgroup).

## 9.3 Stride scheduling: lo mismo, determinista

Variante determinista (también de Waldspurger): cada proceso tiene un **stride** (zancada) inversamente proporcional a sus tickets (p. ej. 10000/tickets) y un contador **pass**. Regla: corre siempre el de menor pass; al terminar su rodaja, pass += stride.

- A (100 tickets, stride 100) avanza despacio su pass → corre a menudo; C (50 tickets, stride 200) la mitad. Reparto **exacto** en cada ciclo, sin varianza.
- Debilidad frente a la lotería: **estado global**. Un proceso que llega nuevo, ¿con qué pass entra? (Con pass=0 monopolizaría hasta alcanzar a los demás; hay que inicializarlo al mínimo/actual — contabilidad delicada que la lotería no necesita.)

## 9.4 CFS: el reparto proporcional en producción (Linux 2007–2023)

El **Completely Fair Scheduler** de Linux lleva la idea a escala industrial, con eficiencia como obsesión (el scheduling consume ~5% del ciclo de CPU en flotas enormes; cada nanosegundo cuenta).

- Cada hilo acumula **vruntime**: su tiempo de CPU ejecutado, ponderado. Corre siempre el hilo con **menor vruntime** — el más "atrasado" en el reparto justo.
- No hay rodaja fija: el tiempo se divide dinámicamente — `sched_latency` (~48 ms como objetivo de periodo) / nº de hilos listos, con suelo `min_granularity` (~6 ms) para no trocear en exceso (amortización otra vez).
- **nice → pesos**: la prioridad clásica de Unix (−20..+19) se traduce a pesos multiplicativos (cada nivel de nice ≈ ×1.25 de CPU); el vruntime avanza *más despacio* para los hilos con más peso — exactamente los tickets de la lotería con otra contabilidad.
- Estructura de datos: los hilos listos viven en un **árbol rojo-negro ordenado por vruntime** → elegir al siguiente es O(1) (el nodo más a la izquierda), insertar O(log n).
- Detalles finos: los hilos que despiertan de dormir reciben un vruntime = mínimo actual (para no monopolizar tras dormir mucho — el problema del pass de stride, resuelto igual); herencia de vruntime en fork; heurísticas de cachés y NUMA (cap. 10).

## 9.5 ¿Cuándo proporcional y cuándo MLFQ?

Proporcional brilla cuando el reparto *es* el objetivo: virtualización (VMware usó shares al estilo lotería), contenedores (`cpu.weight` de cgroups **es** un reparto proporcional jerárquico — la ticket currency hecha sistema), datacenters. En un portátil de escritorio la asignación de tickets no tiene respuesta natural y la E/S complica el reparto — por eso los SO generalistas mezclan ideas.

## Ampliación (no está en OSTEP)

- **EEVDF** (Earliest Eligible Virtual Deadline First, Stoica & Abdel-Wahab 1996) sustituyó a CFS en Linux 6.6 (2023): mantiene el reparto proporcional pero añade *deadlines virtuales* que dan mejor latencia a los hilos que piden poco y a menudo — corrige con teoría lo que CFS parcheaba con heurísticas. Escrito por Peter Zijlstra.
- **sched_ext** (Linux 6.12+, 2024): permite cargar schedulers completos como programas BPF — la separación mecanismo/política del cap. 4 llevada al extremo: la política ahora es un plugin.
- La jerarquía de cgroups v2 hace el reparto **anidado**: primero entre grupos por peso, luego dentro de cada grupo — así reparte CPU Kubernetes entre pods.

## Referencias

- OSTEP cap. 9.
- Waldspurger & Weihl, *Lottery Scheduling*, OSDI 1994; *Stride Scheduling*, MIT TR-528, 1995.
- Stoica & Abdel-Wahab, *Earliest Eligible Virtual Deadline First*, 1996.
- Documentación del kernel: `Documentation/scheduler/sched-design-CFS.rst` y `sched-eevdf.rst`.
