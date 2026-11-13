# 33 · Concurrencia basada en eventos

> Apuntes propios sobre el capítulo 33 de OSTEP (*Event-based Concurrency*). Fuente: https://ostep.org

## 33.1 La alternativa a los hilos

Motivación doble: los hilos son difíciles (todo el catálogo del cap. 32) y ceden el control del entrelazado al scheduler. La concurrencia **por eventos** propone otra estructura, dominante en servidores (nginx, Node.js, Redis):

**El bucle de eventos**:

```c
while (1) {
    eventos = esperar_eventos();       // ¿qué ha pasado? (llegó conexión, hay datos...)
    for (e in eventos)
        procesar(e);                   // ejecutar su manejador, hasta terminar
}
```

**Un solo hilo**; cada manejador corre **hasta completarse** (run-to-completion). No hay entrelazado arbitrario → no hay locks, no hay carreras entre manejadores: el *scheduling lo decide la aplicación* (qué evento procesa y cuándo).

## 33.2 La maquinaria: select/poll → epoll

El SO provee la espera múltiple: `select()`/`poll()` reciben un conjunto de descriptores y devuelven cuáles están listos para leer/escribir sin bloquear (con timeout opcional — que permite hacer trabajo periódico). Sobre esto se construye cualquier servidor de eventos: sockets en modo no bloqueante + un bucle que atiende lo que esté listo.

## 33.3 Regla número uno: no bloquear

Como solo hay un hilo, **un manejador que se bloquea congela el sistema entero** (y un manejador que computa mucho, también). De aquí las dos complicaciones clásicas:

**a) E/S de disco**: históricamente sin variante no-bloqueante → **E/S asíncrona** (AIO): lanzas la operación y sigues; el SO te avisa al completarla (o consultas). Donde no hay AIO completa, el apaño híbrido: un *pool* de hilos solo para las operaciones bloqueantes (lo que hace libuv/Node.js con el disco).

**b) Estado partido (manual stack management)**: en hilos, el estado vive en la pila ("leo fichero, *luego* escribo al socket" es una línea tras otra). En eventos, entre el "leo" y el "luego" hay que **empaquetar a mano el contexto** (el socket destino, etc.) en una **continuación** que el manejador de completado recuperará. Es la queja histórica contra los eventos: el control de flujo explota en callbacks.

## 33.4 Límites del modelo

- **Multi-CPU**: un bucle = un núcleo. Para usar N núcleos, N bucles (procesos/hilos) → vuelven los problemas de sincronización entre ellos (aunque sea a granularidad gruesa).
- **Paginación implícita**: un fallo de página bloquea al hilo único aunque tu código "no bloquee" — bloqueo invisible e inevitable.
- **Evolución de APIs**: una rutina que antes no bloqueaba puede pasar a bloquear en una versión nueva y arruinar el bucle — fragilidad de mantenimiento.
- La vieja discusión "¿hilos o eventos?" (von Behren vs Ousterhout) sigue viva; la síntesis moderna: eventos para E/S masiva, hilos/pools para CPU, y azúcar sintáctico para el estado (siguiente sección).

## Ampliación (no está en OSTEP)

- La maquinaria real actual: `epoll` (Linux, O(1), edge/level-triggered) y `kqueue` (BSD/macOS) sustituyen a select/poll (O(n) por llamada); lo último es **io_uring** (Linux 5.1+): anillos de envío/completado compartidos con el kernel — AIO de verdad para *todo* (disco y red) con mínimas syscalls. Nota curiosa: los anillos de io_uring son primos de las virtqueues de virtio (`../virtualizacion.md` §5.6).
- El problema del "estado partido" lo resolvió el lenguaje: **async/await** (JS, Python, Rust, C#) hace que el compilador construya la continuación por ti — escribes código "secuencial" que se trocea en un autómata de estados sobre un bucle de eventos. Los hilos verdes (Go) son la otra síntesis: pila real barata + scheduler N:M (ver `../virtualizacion.md` §2.4).
- Arquitecturas reales: nginx (procesos worker × bucle epoll cada uno), Redis (un bucle, por eso una operación cara bloquea todo el servidor — y por eso Redis 6 añadió hilos de E/S), Node.js (bucle + thread pool de libuv).

## Referencias

- OSTEP cap. 33.
- Pai, Druschel & Zwaenepoel, *Flash: An Efficient and Portable Web Server*, USENIX 1999.
- Ousterhout, *Why Threads Are a Bad Idea (for most purposes)*, 1996; von Behren et al., *Why Events Are a Bad Idea*, HotOS 2003 — el debate completo.
- `man 7 epoll`; Axboe, *Efficient IO with io_uring*, 2019.
