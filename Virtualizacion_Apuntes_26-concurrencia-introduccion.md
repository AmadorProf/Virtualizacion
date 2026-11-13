# 26 · Concurrencia: introducción

> Apuntes propios sobre el capítulo 26 de OSTEP (*Concurrency: An Introduction*). Fuente: https://ostep.org

## 26.1 Qué es un hilo y por qué usarlo

Un **hilo** es un punto de ejecución dentro de un proceso: su propio PC, registros y **pila**, pero **compartiendo** el espacio de direcciones (código, datos, heap) con los demás hilos del proceso. Un proceso multihilo tiene, pues, varias pilas en su espacio de direcciones (una por hilo).

El cambio entre hilos es un context switch sin cambio de tablas de páginas (no se toca CR3) — más barato que entre procesos. El SO guarda el estado por hilo en TCBs (thread control blocks).

Dos razones para usar hilos:

1. **Paralelismo**: explotar varias CPUs para una misma tarea (dividir un array entre 4 núcleos).
2. **Solapamiento de E/S y cómputo**: mientras un hilo espera red/disco, otros avanzan — el mismo principio del solapamiento del cap. 4, dentro de un proceso. (Los servidores usan hilos por esto tanto como por paralelismo.)

## 26.2 El problema: datos compartidos

El ejemplo canónico: dos hilos hacen `contador++` un millón de veces cada uno. Resultado esperado: 2.000.000. Resultado real: distinto en cada ejecución, y casi siempre menor. ¿Por qué? Porque `contador++` **no es una instrucción**, son tres:

```asm
mov contador, %eax    ; cargar
add $1, %eax          ; sumar
mov %eax, contador    ; guardar
```

Si el timer interrumpe al hilo A entre cargar y guardar, y B ejecuta su incremento completo, al reanudarse A **sobrescribe** el valor de B: dos incrementos, efecto de uno. El scheduler decide cuándo interrumpe → el resultado es **no determinista**.

## 26.3 Vocabulario esencial

- **Condición de carrera (race condition / data race)**: el resultado depende del orden temporal de ejecución de los hilos.
- **Sección crítica**: fragmento de código que accede a datos compartidos y **no debe ser ejecutado por más de un hilo a la vez**.
- **Exclusión mutua (mutual exclusion)**: la propiedad que lo garantiza — de ahí "mutex".
- **Atomicidad**: "todo o nada" — una secuencia que se ejecuta como unidad indivisible, sin interrupción visible. Lo que pedimos para la sección crítica.

Los términos vienen de Dijkstra (años 60; *Cooperating Sequential Processes* y su paper de 1965): el problema es tan viejo como la multiprogramación — los primeros "hilos" con memoria compartida fueron el SO mismo y sus interrupciones.

## 26.4 El plan: primitivas de sincronización

El hardware no va a darnos "sección crítica atómica de tamaño arbitrario" como instrucción; da **primitivas mínimas** (instrucciones atómicas simples, cap. 28) sobre las que, con ayuda del SO, se construyen las herramientas de sincronización: **locks** (cap. 28) para exclusión mutua, y **variables de condición** (cap. 30) / **semáforos** (cap. 31) para el otro problema — **esperar/avisar**: un hilo debe dormir hasta que otro complete algo (p. ej. el padre esperando al hijo, la cola productor/consumidor).

Nota de porqués: esto se estudia en un curso de SO porque el SO fue el primer programa concurrente, y porque el SO participa en las primitivas (dormir/despertar hilos).

## Ampliación (no está en OSTEP)

- Detección automática de carreras: **ThreadSanitizer** (`gcc/clang -fsanitize=thread`) señala data races en ejecución; imprescindible en C/C++ moderno. `helgrind` (valgrind) es la alternativa clásica.
- El estándar C11/C++11 declaró los data races **comportamiento indefinido**: el compilador puede asumir que no existen y optimizar en consecuencia — una carrera no solo da valores raros: puede romper el programa de formas creativas. La respuesta del lenguaje son los *atomics* con **modelo de memoria** formal (`std::atomic`, `memory_order_*`).
- Perspectiva de diseño: Rust convierte las carreras de datos en **error de compilación** (ownership + `Send`/`Sync`); Go recomienda "no comuniques compartiendo memoria; comparte comunicando" (channels — descendientes de CSP de Hoare, ver cap. 33).

## Referencias

- OSTEP cap. 26.
- Dijkstra, *Solution of a Problem in Concurrent Programming Control*, CACM 1965; *Cooperating Sequential Processes*, 1968.
- Serebryany et al., *ThreadSanitizer*, 2009.
- Boehm & Adve, *Foundations of the C++ Concurrency Memory Model*, PLDI 2008.
