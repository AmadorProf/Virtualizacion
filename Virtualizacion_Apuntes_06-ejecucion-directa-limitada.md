# 6 · Mecanismo: ejecución directa limitada

> Apuntes propios sobre el capítulo 6 de OSTEP (*Mechanism: Limited Direct Execution*). Fuente: https://ostep.org

Este capítulo responde al **cómo** de la virtualización de CPU. Hay dos problemas simultáneos: rendimiento (no puede ser lento virtualizar) y control (el SO debe seguir mandando). La solución de todos los SO modernos se resume en el nombre del capítulo.

## 6.1 Ejecución directa… limitada

**Ejecución directa**: el programa corre *directamente sobre la CPU real*, a velocidad nativa — nada de interpretar sus instrucciones. Rápido, pero plantea dos preguntas:

1. ¿Cómo evitamos que haga lo que no debe (E/S directa, leer datos ajenos)?
2. ¿Cómo recupera el SO la CPU si el programa no la suelta?

Las respuestas son las dos "limitaciones": modos de CPU + interrupciones de temporizador.

## 6.2 Problema 1: operaciones restringidas → modos y syscalls

La CPU ofrece (mínimo) dos modos:

- **Modo usuario**: las instrucciones privilegiadas (E/S, configurar memoria, deshabilitar interrupciones) están prohibidas; intentarlas lanza una excepción y el SO normalmente mata al proceso.
- **Modo kernel**: todo permitido. Aquí corre el SO.

¿Y si el proceso *necesita* algo privilegiado (leer un fichero)? Ejecuta una **syscall** mediante una instrucción de **trap**:

```
  usuario                          kernel
 ─────────                        ─────────
  open("f")                          
    → trap ───────────────────→  trap handler
                                  valida argumentos
                                  hace el trabajo (privilegiado)
    continúa  ←── return-from-trap
```

Detalles que importan:

- La instrucción de trap **eleva el privilegio y salta a un punto fijo que eligió el kernel**, no el usuario. ¿Cómo sabe la CPU adónde saltar? Por la **tabla de traps/interrupciones** (IDT en x86), que el kernel instala **en el arranque** (instrucción privilegiada). El usuario solo puede pedir *un número de servicio* (el nº de syscall en un registro), nunca una dirección.
- Al entrar al kernel, el hardware guarda registros del proceso (en la **pila de kernel** de ese proceso) para poder reanudarlo con return-from-trap.
- El kernel debe **desconfiar de todos los argumentos** (punteros del usuario, etc.): validar o ser hackeado.

Sobre esta frontera se apoya todo lo demás — y es exactamente el patrón que los hipervisores repiten un nivel más abajo (VM exit; ver `../virtualizacion.md` §5).

## 6.3 Problema 2: recuperar la CPU → el temporizador

Si un proceso entra en un bucle infinito y nunca hace syscalls, ¿cómo vuelve a mandar el SO? Dos enfoques históricos:

- **Cooperativo**: confiar en que los procesos cedan la CPU (syscall `yield`, o al hacer cualquier syscall). Lo usaron los primeros Macintosh. Problema evidente: un proceso colgado o malicioso secuestra la máquina; solo queda reiniciar.
- **Apropiativo (preemptive)**: la solución real — el **timer interrupt**. El kernel programa en el arranque un temporizador hardware que interrumpe cada pocos milisegundos. La interrupción detiene al proceso *haga lo que haga*, guarda su estado y entra al kernel. Recuperado el control, el scheduler decide: ¿sigue este o cambio?

La interrupción de temporizador es la garantía física de que **el SO siempre recupera la CPU**. Sin ella no hay multitarea robusta.

## 6.4 El cambio de contexto (context switch)

Si el scheduler decide cambiar de proceso A a B:

1. (Hardware, al entrar la interrupción) guarda los registros de usuario de A en la pila de kernel de A.
2. (Software) el kernel guarda los registros de kernel de A en su PCB y **restaura los de B desde su PCB** — incluido el puntero de pila: a partir de aquí el kernel está "siendo" B.
3. Return-from-trap restaura los registros de usuario de B → B continúa donde lo dejaron, sin enterarse.

Hay pues **dos guardados distintos**: usuario→kernel (lo hace el hardware en el trap) y kernel-de-A→kernel-de-B (lo hace el SO, típicamente una rutina corta en ensamblador — `swtch` en xv6).

**Coste**: el directo son µs (guardar/restaurar + trabajo del scheduler); el indirecto suele pesar más — cachés, TLB y predictores llenos de datos del proceso anterior. Herramienta clásica de medición: **lmbench**.

## 6.5 Casos límite

- ¿Interrupción durante una interrupción? Los kernels deshabilitan interrupciones brevemente durante el manejo (con cuidado: demasiado tiempo pierde eventos) y usan locks para proteger sus estructuras en multiprocesador — anticipo de la parte de concurrencia.

## Ampliación (no está en OSTEP)

- En x86-64 moderno el trap de syscall es la instrucción `syscall` (más rápida que el viejo `int 0x80`); el kernel fija el destino en el MSR `LSTAR`. Puedes ver la frontera en acción con `strace -T` (tiempo por syscall).
- Coste real de una syscall trivial hoy: ~100–300 ns… que las mitigaciones de Spectre/Meltdown (KPTI, 2018) llegaron a multiplicar — el aislamiento cuesta. Referencia: Gregg, *KPTI performance*.
- **vDSO**: Linux evita el trap por completo en syscalls muy frecuentes de solo lectura (`gettimeofday`, `clock_gettime`) mapeando una página del kernel en el proceso — la excepción que confirma la regla.
- El *tick* del temporizador en Linux es configurable (CONFIG_HZ: 100–1000 Hz) y hoy es *tickless* (NO_HZ) cuando la CPU está ociosa o con un solo hilo listo, por energía y latencia.

## Referencias

- OSTEP cap. 6.
- Código de xv6: `trap.c`, `swtch.S`, `syscall.c` — el capítulo entero en ~300 líneas de código real.
- McVoy & Staelin, *lmbench: Portable tools for performance analysis*, USENIX 1996.
- Intel SDM vol. 3 (caps. de interrupciones y modo de sistema) para el detalle x86.
