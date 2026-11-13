# 20 · Tablas de páginas más pequeñas

> Apuntes propios sobre el capítulo 20 de OSTEP (*Paging: Smaller Tables*). Fuente: https://ostep.org

## 20.1 El problema

La tabla lineal es un array con una entrada por **cada página virtual posible**, exista o no. Un espacio de 32 bits con páginas de 4 KiB → 2^20 entradas × 4 B = 4 MiB *por proceso*; en 64 bits, cifras absurdas. Pero un proceso típico usa una fracción minúscula y **dispersa** de su espacio (código abajo, heap tras él, pila arriba, huecos gigantes en medio). Objetivo: pagar tabla solo por lo usado.

## 20.2 Intentos previos

**Páginas más grandes**: menos entradas, pero fragmentación interna → no como solución general (aunque las huge pages existen por otra razón: la TLB, cap. 19).

**Híbrido segmentación+paginación**: una tabla de páginas *por segmento* (código/heap/pila), cada una del tamaño justo (el registro límite del segmento indica cuántas páginas válidas). Elimina las entradas del gran hueco central… pero hereda los males de la segmentación (regiones dispersas dentro de un segmento vuelven a desperdiciar; las tablas ahora son de tamaño variable → fragmentación externa para *ellas*).

## 20.3 La solución dominante: tablas multinivel

Idea: **paginar la propia tabla de páginas**. Se trocea la tabla lineal en trozos del tamaño de una página; un nivel superior — el **directorio de páginas** — tiene una entrada por trozo que dice si ese trozo contiene *algo* válido y dónde está. Los trozos completamente inválidos **no se materializan**: el hueco de gigas entre heap y pila cuesta unas pocas entradas de directorio marcadas inválidas.

```
        VPN = [ índice directorio | índice tabla ]   [offset]
                       │                 │
  PDBR ─→ directorio ──┴→ trozo de tabla ┴→ PFN
           (1 página)      (1 página)
```

- **Coste espacial**: proporcional al espacio *usado* (con la pequeña sobrecarga del directorio). Además cada pieza es exactamente una página → asignarlas es trivial, sin contigüidad física exigida (la tabla lineal grande la exigía).
- **Coste temporal**: el *time-space trade-off* del capítulo — un miss de TLB ahora requiere **un acceso por nivel** (2 en el ejemplo, 4–5 en x86-64 real). Pagas tiempo en el caso miss por ahorrar espacio siempre.
- Con espacios enormes, el propio directorio se pagina también → **más niveles**: x86-64 usa 4 (PGD→PUD→PMD→PTE; 9+9+9+9+12 bits) y opcionalmente 5 (LA57). El esquema del cap. 18 de `../virtualizacion.md` §3.2 es exactamente este.

## 20.4 Alternativas

- **Tablas invertidas**: una única tabla global con una entrada **por marco físico** (qué proceso/página lo ocupa). Tamaño proporcional a la RAM, no a los espacios virtuales — elegante, pero buscar "¿dónde está la página virtual X del proceso P?" exige una tabla hash sobre ella. Lo usaron PowerPC y PA-RISC.
- **Swapear las tablas**: si ni así caben, las tablas mismas pueden ir a disco (lo hacía VAX/VMS, cap. 23).

## Ampliación (no está en OSTEP)

- El page walk de 4 niveles se mitiga con **paging-structure caches** (cachean los niveles altos: la mayoría de walks solo leen el último nivel) y con huge pages (2 MiB corta el walk un nivel antes: 3 accesos; 1 GiB, dos).
- En hipervisores el multinivel se **compone**: guest 4 niveles × EPT 4 niveles = hasta 24 accesos por miss (`../virtualizacion.md` §5.5) — el trade-off espacio/tiempo elevado al cuadrado.
- Linux gestiona 5 niveles *siempre* en el código (PGD/P4D/PUD/PMD/PTE) y "pliega" los que el hardware no tiene — truco de portabilidad para soportar x86-64, ARM64, RISC-V con el mismo código (`Documentation/mm/page_tables.rst`).

## Referencias

- OSTEP cap. 20.
- Talluri, Hill & Khalidi, *A New Page Table for 64-bit Address Spaces*, SOSP 1995 — comparación clásica de estructuras (incluye invertidas y guarded page tables).
- Intel SDM vol. 3 cap. 4; manual ARM ARM (translation tables) para el contraste.
