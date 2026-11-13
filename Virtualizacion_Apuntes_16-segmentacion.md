# 16 · Segmentación

> Apuntes propios sobre el capítulo 16 de OSTEP (*Segmentation*). Fuente: https://ostep.org

## 16.1 Generalizar base y límite

El problema del cap. 15: reservar el espacio de direcciones *entero y contiguo* desperdicia el gran hueco entre heap y pila. Idea de la segmentación: **un par base/límite por cada región lógica** (segmento) — código, heap, pila — y colocar cada segmento **independientemente** en RAM física. Solo se reserva lo que cada segmento usa; el hueco virtual no ocupa nada físico.

```
 RAM física:
  ┌────────┐
  │ código │← base_código
  ├────────┤     (los tres segmentos, dispersos
  │  pila  │← base_pila        y compactos)
  ├────────┤
  │  heap  │← base_heap
  └────────┘
```

## 16.2 ¿A qué segmento pertenece una dirección?

Dos enfoques:

- **Explícito**: los bits altos de la dirección virtual seleccionan el segmento; el resto es el desplazamiento (offset) dentro de él. Con 3 segmentos, 2 bits: `01|0110100` → segmento 1 (heap), offset 0x34 → física = base_heap + 0x34, comprobando offset < límite_heap. Pega: partir el espacio en cuadrantes fijos limita el tamaño de cada segmento (y desperdicia un cuadrante si hay 3 segmentos y 4 combinaciones).
- **Implícito**: el hardware deduce el segmento por el origen del acceso (fetch → código; relativo a SP → pila; resto → heap).

**La pila complica**: crece hacia abajo. Hace falta un bit por segmento que indique el sentido de crecimiento y una aritmética de traducción ligeramente distinta (offsets "negativos" desde el tope).

## 16.3 Compartición y protección

Con **bits de protección** por segmento (lectura/escritura/ejecución) aparece un beneficio nuevo: **compartir segmentos de código** entre procesos (el mismo segmento físico de solo-lectura mapeado en muchos procesos), con la ilusión intacta. La comprobación de permisos se hace junto a la de límites; violarla → excepción.

## 16.4 El problema fatal: fragmentación externa

El SO ahora gestiona huecos de **tamaños variables** en RAM. Al crear y destruir procesos, la memoria libre queda picada en huequitos: puede haber 24 MiB libres en total y no caber un segmento de 20 MiB porque **ningún hueco es contiguo suficiente**. Es la **fragmentación externa**.

Remedios paliativos:

- **Compactar**: parar procesos, mover segmentos para juntar los huecos, actualizar bases. Carísimo (copiar memoria masivamente) y hostil a heaps que crecen.
- **Mejores algoritmos de asignación** (best-fit, worst-fit, buddy… cap. 17): ayudan, pero *ningún algoritmo elimina* la fragmentación externa con tamaños variables.

Además, la segmentación sigue siendo **gruesa**: si el heap está casi vacío por dentro, igualmente su parte usada+dispersa debe estar completa en RAM. La solución a ambas cosas — trocear en unidades **fijas y pequeñas** — es la paginación (cap. 18).

## Ampliación (no está en OSTEP)

- x86 de 32 bits tenía segmentación completa en hardware (GDT/LDT, selectores CS/DS/SS); casi nadie la usó de verdad encima de la paginación, y **x86-64 la eliminó de facto** (bases forzadas a 0, salvo FS/GS que sobreviven como punteros baratos a datos por-hilo — TLS — y per-CPU en el kernel). La historia falló a favor de la paginación pura.
- El término *segmentation fault* es el fósil lingüístico de esta época: hoy casi siempre señala una violación de paginación, no de segmentos.
- La idea de "regiones lógicas con permisos" sobrevive en otro plano: las **VMAs** de Linux (`/proc/pid/maps`) son exactamente eso, implementadas *sobre* paginación.

## Referencias

- OSTEP cap. 16.
- Dennis, *Segmentation and the Design of Multiprogrammed Computer Systems*, JACM 1965 — el paper fundacional (Multics).
- Intel SDM vol. 3, cap. de segmentación — para ver el fósil x86 en detalle.
