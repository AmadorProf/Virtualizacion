# 8 · Planificación: la cola multinivel con retroalimentación (MLFQ)

> Apuntes propios sobre el capítulo 8 de OSTEP (*Scheduling: The Multi-Level Feedback Queue*). Fuente: https://ostep.org

## 8.1 El problema

Queremos a la vez buen turnaround (favorecer trabajos cortos, como STCF) y buena respuesta (interactividad, como RR)… **sin conocer la duración de nada**. La MLFQ (Corbató, sistema CTSS del MIT, 1962 — le valió el Turing) resuelve el dilema **aprendiendo del comportamiento observado**: el pasado reciente predice el futuro.

## 8.2 La estructura

Varias colas, cada una con un nivel de **prioridad**. Reglas básicas:

- **R1**: si prioridad(A) > prioridad(B), corre A.
- **R2**: si prioridad(A) = prioridad(B), corren en round-robin.

La inteligencia está en cómo se mueve cada trabajo entre colas — la **retroalimentación**:

- **R3**: todo trabajo nuevo entra en la **cola más alta**. (Optimismo: si resulta ser corto, terminará enseguida, imitando a SJF; si es largo, ya bajará.)
- **R4 (primera versión)**: si un trabajo consume su rodaja entera, baja un nivel; si suelta la CPU antes (E/S), se queda donde está.

Efecto: los intensivos en CPU se hunden hacia abajo; los interactivos (ráfagas cortas + E/S) flotan arriba. MLFQ **aproxima SJF sin oráculo**.

## 8.3 Los tres problemas y sus parches

**a) Inanición (starvation).** Con bastantes interactivos arriba, los de abajo no corren jamás. Parche — **R5, el impulso de prioridad**: cada S segundos, **todos** los trabajos suben a la cola más alta. Garantiza progreso y readapta a los procesos que cambian de fase (un batch que se vuelve interactivo). El valor de S es magia negra ("constante vudú" en el libro): grande → inanición; pequeño → los interactivos pierden su ventaja.

**b) Trampas (gaming).** Con la R4 ingenua, un proceso listillo hace una E/S de mentira al 99% de su rodaja → nunca baja de cola y monopoliza. Parche — **R4 mejorada, contabilidad total**: el scheduler acumula el tiempo de CPU usado *en ese nivel* independientemente de cuántas veces lo suelte; agotado el presupuesto del nivel, baja. Ceder la CPU ya no borra el historial.

**c) Cambios de fase**: cubierto también por el impulso periódico (R5).

## 8.4 Reglas finales

1. Prioridad mayor corre primero.
2. Igual prioridad → round-robin.
3. Trabajo nuevo → cola más alta.
4. Consumido el presupuesto de un nivel (sume o no rodajas completas) → baja un nivel.
5. Cada S segundos, todos a la cola más alta.

## 8.5 Afinación

Parámetros libres: nº de colas, rodaja por cola (típico: cortas arriba ~10 ms, largas abajo ~cientos de ms — los de abajo son batch, mejor amortizar), presupuesto por nivel, S. Solaris (TS class) lo hacía con tablas editables; otros sistemas usan fórmulas matemáticas sobre el uso reciente (los *decay-usage schedulers* como el de FreeBSD 4.3+). Algunos SO reservan además las colas altas para el sistema, y dejan al usuario dar *consejos* (`nice`).

## Ampliación (no está en OSTEP)

- Windows usa un scheduler de este estilo (32 niveles, con *boosts* de prioridad tras E/S y para la ventana en primer plano). macOS/XNU también es multinivel con bandas (throughput/utility/background).
- **Linux no usa MLFQ**: usó O(1) (2002, con heurísticas de interactividad muy MLFQ-style) y desde 2007 CFS — reparto proporcional (cap. 9) — y desde 6.6 EEVDF. La historia real del scheduling de Linux es un péndulo entre heurísticas (MLFQ-ish) y equidad matemática.
- La inanición no es teórica: los *priority inversion* + colas bajas causaron el célebre fallo del Mars Pathfinder (1997) — combinación con locks, ver cap. 28/32.
- MLFQ es un ejemplo temprano de **aprendizaje online** con presupuestos; la misma idea (clasificar por comportamiento observado) reaparece en los schedulers de E/S y en los de GPU.

## Referencias

- OSTEP cap. 8.
- Corbató et al., *An Experimental Time-Sharing System*, AFIPS 1962 (CTSS).
- Epema, *Decay-usage scheduling in multiprocessors*, ACM TOCS 1998.
- Russinovich et al., *Windows Internals* (scheduler de Windows).
