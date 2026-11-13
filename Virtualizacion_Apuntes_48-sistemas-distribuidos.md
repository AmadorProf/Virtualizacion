# 48 · Sistemas distribuidos: introducción

> Apuntes propios sobre el capítulo 48 de OSTEP (*Distributed Systems*). Fuente: https://ostep.org

## 48.1 El cambio de mentalidad

Un sistema distribuido: varias máquinas cooperando por red para dar un servicio. La diferencia filosófica con todo lo anterior: dentro de una máquina, el fallo era excepcional; con miles de máquinas, **el fallo es el estado normal** (siempre hay algún disco, máquina o cable roto). El objetivo paradójico: construir un sistema que *parezca* que nunca falla a partir de piezas que fallan constantemente.

Temas transversales: fallo (¿parcial!), **comunicación** (el foco de este capítulo), rendimiento (latencia de red ≫ RAM), seguridad, y escala.

## 48.2 La comunicación es intrínsecamente no fiable

Los paquetes se pierden: buffers que se desbordan en routers y NICs, cables defectuosos, congestión. La base de todo el stack (IP, UDP) asume pérdida y no la oculta. Sobre esa base hay dos filosofías:

**a) Abstracción no fiable (UDP)**: datagramas — rápido, simple; los paquetes pueden perderse, duplicarse o llegar desordenados; la aplicación se apaña. Con **checksums** al menos se detecta corrupción.

**b) Comunicación fiable (TCP y equivalentes)** construida con dos piezas:

- **ACK**: el receptor confirma cada mensaje recibido.
- **Timeout + retransmisión**: si no llega el ACK a tiempo, reenviar (con **backoff exponencial** para no ahogar la red congestionada).
- El problema resultante: el reenvío puede **duplicar** mensajes (se perdió el ACK, no el mensaje) → hacen falta **números de secuencia** para detectar duplicados y descartar. TCP empaqueta todo esto (más control de congestión y ventanas).

Regla mental clave: "no llegó ACK" NO distingue "se perdió mi mensaje" de "se perdió su respuesta" — la ambigüedad fundamental de lo distribuido, que reaparece en cada nivel (¿el servidor ejecutó mi operación o no?).

## 48.3 Abstracciones de comunicación

¿Qué interfaz dar al programador?

- **Memoria compartida distribuida (DSM)**: hacer que varias máquinas "compartan" espacio de direcciones vía page faults por red. Fracasó: los fallos parciales la rompen (¿qué haces con las páginas de una máquina muerta?) y el rendimiento es opaco (un load puede costar ms). Casi nadie la usa.
- **RPC (Remote Procedure Call)**: la ganadora — hacer que llamar a un servicio remoto *parezca* una llamada a función. Dos componentes:
  - **Generador de stubs**: dado el prototipo, genera el código cliente (empaquetar argumentos = *marshaling*, enviar, esperar, desempaquetar retorno) y el esqueleto servidor (lo inverso). Automatiza lo tedioso y propenso a errores.
  - **Runtime**: resuelve nombres, elige transporte (a menudo UDP + fiabilidad propia, o TCP), gestiona timeouts/reintentos, fragmentación, concurrencia en el servidor (pool de hilos), y las semánticas de fallo.

Detalles con miga: argumentos con punteros (¿copiar la estructura entera?), llamadas que tardan (ACK explícito "en ello" para distinguir lento de muerto), byte order (big vs little endian — resuelto en el protocolo, p. ej. XDR), y **procedimientos idempotentes** cuando la semántica es "al menos una vez".

La advertencia final del capítulo: RPC *parece* una función pero **no lo es** — puede fallar de formas que una función local no puede, y cuesta 4-6 órdenes de magnitud más. Ocultarlo del todo es una mentira peligrosa (el famoso paper *A Note on Distributed Computing*: la transparencia total es imposible).

## Ampliación (no está en OSTEP)

- RPC hoy: **gRPC** (protobuf + HTTP/2), Thrift, JSON/REST — los descendientes directos del modelo stub+runtime; los mismos problemas de timeouts, reintentos e idempotencia, ahora con nombres como "circuit breakers" y "retry budgets".
- Lo que OSTEP deja fuera y sigue el itinerario natural: **consenso** (Paxos, **Raft** — raft.github.io, legible), relojes lógicos de **Lamport** (*Time, Clocks…*, 1978, quizá el paper más influyente del área), teorema **CAP**, y las falacias del computing distribuido (Deutsch). Curso recomendado: MIT 6.824/6.5840 (labs con Raft en Go, gratis online).
- Los números de secuencia + reintentos + idempotencia del capítulo son literalmente el examen de entrevista de sistemas: "diseña un sistema de pagos que no cobre dos veces" = este capítulo.

## Referencias

- OSTEP cap. 48.
- Birrell & Nelson, *Implementing Remote Procedure Calls*, ACM TOCS 1984 — el paper de RPC.
- Waldo et al., *A Note on Distributed Computing*, 1994.
- Lamport, *Time, Clocks, and the Ordering of Events in a Distributed System*, CACM 1978.
- Ongaro & Ousterhout, *In Search of an Understandable Consensus Algorithm* (Raft), USENIX ATC 2014.
