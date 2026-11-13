# 7 · Planificación: introducción

> Apuntes propios sobre el capítulo 7 de OSTEP (*Scheduling: Introduction*). Fuente: https://ostep.org

Con el mecanismo resuelto (cap. 6), toca la **política**: ¿a qué proceso le doy la CPU ahora? Este capítulo construye las políticas clásicas partiendo de supuestos irreales y quitándolos uno a uno — un método didáctico que merece imitarse.

## 7.1 Supuestos iniciales y métricas

Supuestos de partida (todos falsos, se irán retirando): los trabajos duran lo mismo, llegan a la vez, corren hasta acabar, no hacen E/S, y **conocemos su duración**.

Dos métricas en tensión:

- **Turnaround** (tiempo de retorno) = T_finalización − T_llegada. Métrica de *rendimiento*.
- **Tiempo de respuesta** = T_primera_ejecución − T_llegada. Métrica de *interactividad* (nadie quiere teclear y esperar 10 s al eco).

También importa la **equidad** (fairness); rendimiento y equidad suelen estar reñidos.

## 7.2 FIFO (FCFS) y el efecto convoy

Ejecutar por orden de llegada. Simple, pero basta que un trabajo largo llegue primero para arruinar a los cortos: llega uno de 100 s y dos de 10 s → turnarounds 100, 110, 120 (media 110). Los cortos hacen cola tras el "camión": **efecto convoy** — el mismo fenómeno de la cola única del supermercado con un carro lleno delante.

## 7.3 SJF y STCF: los cortos primero

- **SJF** (Shortest Job First): ejecutar siempre el trabajo pendiente más corto. Con todos llegando a la vez es **óptimo en turnaround medio** (resultado clásico de teoría de colas). Pero es *no apropiativo*: si el largo ya empezó y los cortos llegan un segundo después, vuelven a esperar el convoy.
- **STCF** (Shortest Time-to-Completion First, = SJF apropiativo): al llegar un trabajo nuevo, si su tiempo restante es menor que el del actual, lo **desaloja**. Óptimo en turnaround incluso con llegadas escalonadas.

Problema doble: el tiempo de respuesta de los que esperan es malo, y — fatal en la práctica — **requiere conocer la duración de los trabajos**, que el SO no conoce.

## 7.4 Round Robin: optimizar la respuesta

**RR** ejecuta cada trabajo una **rodaja de tiempo** (time slice, múltiplo del tick del temporizador) y rota. Con rodaja pequeña, todo el mundo "arranca" enseguida → excelente tiempo de respuesta.

El precio: **turnaround pésimo** (todo el mundo acaba tarde: RR estira a todos hasta el final) — es casi la política más injusta posible *hacia el turnaround* siendo la más justa en reparto instantáneo.

Elección de la rodaja = amortización: cambiar de contexto cuesta (µs directos + cachés frías); con rodaja de 10 ms y cambio de 1 ms se pierde el 10%; con rodaja de 100 ms, el 1%. Regla: **suficientemente larga para amortizar el cambio, suficientemente corta para responder bien**.

Conclusión del par de familias: SJF/STCF optimizan turnaround, RR optimiza respuesta; **cualquier política justa en respuesta será mala en turnaround y viceversa**. No hay almuerzo gratis.

## 7.5 Incorporando la E/S

Los procesos reales se bloquean en E/S constantemente. Truco: tratar **cada ráfaga de CPU entre E/Ss como un "trabajo" independiente**. Un proceso interactivo (ráfagas de 1 ms) se ve como muchos trabajos cortísimos → STCF lo prioriza, y mientras espera su E/S, el trabajo largo usa la CPU: **solapamiento** perfecto de CPU y disco.

## 7.6 El supuesto que no cae: el oráculo

Todo lo bueno de SJF/STCF depende de conocer el futuro. El SO no lo conoce… pero puede **predecirlo con el pasado**: los procesos tienen fases y hábitos. Cómo construir un scheduler que aprende sin oráculo es exactamente el capítulo siguiente (MLFQ).

## Ampliación (no está en OSTEP)

- La predicción clásica de ráfagas usa **media móvil exponencial**: τ_{n+1} = α·t_n + (1−α)·τ_n (aparece en Silberschatz). MLFQ (cap. 8) logra lo mismo estructuralmente, sin fórmula.
- SJF óptimo tiene su formalización en teoría de colas (regla SRPT); Schrage 1968 probó la optimalidad de SRPT/STCF.
- En sistemas reales el turnaround puro casi nunca es la métrica: los servidores optimizan **latencia p99**, los batch optimizan throughput, los móviles energía. Identificar *qué* métrica importa es la mitad del problema de scheduling.

## Referencias

- OSTEP cap. 7.
- Schrage, *A proof of the optimality of the shortest remaining processing time discipline*, Operations Research 1968.
- Kleinrock, *Queueing Systems* — la teoría de colas detrás de estas intuiciones.
