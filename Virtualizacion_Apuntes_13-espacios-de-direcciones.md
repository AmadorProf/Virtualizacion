# 13 · La abstracción: espacios de direcciones

> Apuntes propios sobre el capítulo 13 de OSTEP (*The Abstraction: Address Spaces*). Fuente: https://ostep.org

## 13.1 De dónde venimos

- **Primeros sistemas**: un solo programa en RAM, junto al SO (que era poco más que una biblioteca residente en las primeras direcciones). Sin protección: no hacía falta.
- **Multiprogramación y time sharing**: varios procesos residentes a la vez para aprovechar la máquina (y porque guardar/restaurar la memoria completa a disco en cada cambio era inaceptablemente lento). Nuevo problema: un proceso puede leer o escribir la memoria de otro. La **protección** pasa a ser requisito central.

## 13.2 El espacio de direcciones

La abstracción resultante: cada proceso ve **su propia memoria**, un rango de direcciones que empieza en 0 y es potencialmente enorme, con esta anatomía convencional:

```
 0        ┌──────────────┐
          │    código    │  (estático; también datos globales)
          ├──────────────┤
          │     heap     │  malloc(); crece hacia abajo ↓
          │      ⋮       │
          │   (libre)    │
          │      ⋮       │
          │     pila     │  locales, llamadas; crece hacia arriba ↑
 máx      └──────────────┘
```

Heap y pila se colocan en extremos opuestos para que ambos puedan crecer hacia el hueco central. (Con hilos, la convención se complica: una pila por hilo.)

Lo esencial: **toda dirección que un programa ve es virtual**. Cuando imprime un puntero, el 0x400000 que muestra no es un lugar físico; el SO + hardware lo traducen a la dirección física real, distinta en cada proceso aunque el número virtual coincida.

## 13.3 Objetivos de la memoria virtual

1. **Transparencia**: el programa no debe notar que su memoria es una ilusión; escribe como si tuviera la RAM entera para él.
2. **Eficiencia**: ni mucho tiempo extra (→ hará falta ayuda del hardware: TLBs) ni mucho espacio extra (→ tablas de traducción compactas).
3. **Protección/aislamiento**: ningún proceso puede tocar nada fuera de su espacio; el fallo de uno no contamina a otros ni al SO.

El aislamiento es un principio general: aislar componentes limita el daño mutuo (los microkernels llevan la idea al extremo aislando trozos del propio SO).

## 13.4 El plan de los próximos capítulos

Mecanismo base (traducción por hardware, cap. 15) → esquemas históricos y sus fallos (base/límite, segmentación cap. 16) → gestión de huecos (cap. 17) → la solución moderna (paginación, caps. 18–20) → ir más allá de la RAM física (swap, caps. 21–22) → sistemas reales completos (cap. 23).

## Ampliación (no está en OSTEP)

- Verifica la ilusión en Linux: `cat /proc/self/maps` muestra el espacio real de un proceso — verás código, heap, `[stack]`, y además mmaps de bibliotecas y regiones especiales (`[vdso]`, `[vvar]`) que el dibujo canónico omite.
- **ASLR** (Address Space Layout Randomization): los SO modernos aleatorizan las posiciones de pila, heap y bibliotecas en cada ejecución como defensa anti-exploits — otra ventaja de que todo sea virtual. Ejecuta dos veces `cat /proc/self/maps` y compara.
- En x86-64 el espacio "enorme" son 48 bits útiles (256 TiB, mitad usuario/mitad kernel); las extensiones LA57 llevan a 57 bits (128 PiB) en servidores grandes.

## Referencias

- OSTEP cap. 13.
- Denning, *Virtual Memory*, ACM Computing Surveys 1970 — el survey clásico del área.
- `man 5 proc` (sección `maps`); PaX/ASLR documentación para la parte de seguridad.
