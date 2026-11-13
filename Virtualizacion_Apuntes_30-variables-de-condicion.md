# 30 · Variables de condición

> Apuntes propios sobre el capítulo 30 de OSTEP (*Condition Variables*). Fuente: https://ostep.org

## 30.1 El problema: esperar, no solo excluir

Los locks resuelven la exclusión mutua, pero hay un segundo problema de sincronización: **esperar a que algo sea cierto** (el padre espera a que el hijo termine; el consumidor, a que haya datos). La espera activa sobre un flag funciona pero desperdicia CPU y es frágil: hace falta dormir y ser despertado.

## 30.2 La primitiva

Una **variable de condición (CV)** es una cola de hilos dormidos esperando una condición, con dos operaciones:

- `wait(cv, mutex)`: **suelta el mutex y duerme, atómicamente**; al ser despertado, readquiere el mutex antes de retornar. (La atomicidad soltar+dormir es la esencia: sin ella, el aviso puede llegar en el hueco y perderse.)
- `signal(cv)`: despierta a un hilo dormido en la cv (si lo hay). `broadcast(cv)`: despierta a todos.

## 30.3 Las dos reglas de oro

El capítulo las destila tras enumerar los fallos posibles:

1. **Siempre con el lock cogido** al hacer wait y signal. Wait lo exige por diseño; signal sin lock abre la carrera clásica: el que va a dormirse comprueba la condición, es interrumpido, el otro señala "al vacío", y el primero se duerme para siempre.
2. **Recomprobar la condición con `while`, no `if`** al despertar. Razones: entre el signal y tu despertar, otro hilo pudo consumir la condición (semántica **Mesa**, la de todos los sistemas reales: signal es una *pista*, no una garantía — frente a la semántica Hoare, teórica, de traspaso inmediato); y existen **despertares espurios**.

Corolario: el estado de la condición se guarda en **variables normales protegidas por el mutex** (la CV no "recuerda" señales — contraste con semáforos, cap. 31).

## 30.4 Productor/consumidor (bounded buffer)

El problema canónico (Dijkstra): un buffer de tamaño N, productores que meten y consumidores que sacan. Solución correcta final:

```c
cond_t lleno, vacio; mutex_t m;

// productor                         // consumidor
lock(&m);                            lock(&m);
while (cuenta == N)                  while (cuenta == 0)
    wait(&vacio, &m);                    wait(&lleno, &m);
    poner(item);                         item = sacar();
signal(&lleno);                      signal(&vacio);
unlock(&m);                          unlock(&m);
```

Los dos errores que el capítulo va corrigiendo para llegar aquí:

- Con `if` en lugar de `while` → un tercer hilo se cuela entre el signal y el despertar y consume el dato → el despertado opera sobre buffer vacío.
- Con **una sola CV** → un consumidor puede despertar a *otro consumidor* (en vez de a un productor) y todos acaban dormidos. **Dos CVs**: los productores esperan en una, los consumidores en otra; cada clase señala a la contraria.

## 30.5 Covering conditions

Ejemplo del asignador de memoria: hilos esperan a que haya *suficientes* bytes libres, pero cada uno necesita cantidades distintas — ¿a quién despertar al liberar? Solución conservadora: **broadcast** — despertar a todos y que cada uno recompruebe (el `while` hace el filtro). Correcto siempre; costoso si son muchos. Se llama *covering condition*: cuando no sabes a quién le sirve el cambio, avisa a todos.

## Ampliación (no está en OSTEP)

- Regla práctica sobre señalizar con el lock cogido vs soltarlo antes: señalar con lock es siempre correcto (y lo que hay que enseñar); algunos runtimes optimizan el "hurry up and wait" moviendo el signal fuera — con cuidado extremo.
- pthreads: `pthread_cond_timedwait` (esperar con plazo), y atención a que un `pthread_cond_signal` sin nadie esperando **se pierde** (no queda registrado) — motivo por el que el estado va en variables aparte.
- Implementación en Linux: las CVs de glibc están construidas sobre **futex** (cap. 28), con una máquina de estados sorprendentemente sutil (hubo bugs célebres de despertares perdidos hasta la reescritura de 2016, glibc 2.25).
- Los monitores de Java (`synchronized` + `wait/notify`) y `Condition` de java.util.concurrent son exactamente esto; el paper histórico es Lampson & Redell, *Experience with Processes and Monitors in Mesa* (1980) — de donde viene el nombre "semántica Mesa" y el consejo del `while`.

## Referencias

- OSTEP cap. 30.
- Hoare, *Monitors: An Operating System Structuring Concept*, CACM 1974.
- Lampson & Redell, *Experience with Processes and Monitors in Mesa*, CACM 1980.
- `man 3 pthread_cond_wait`.
