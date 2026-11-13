# 44 · SSDs basadas en flash

> Apuntes propios sobre el capítulo 44 de OSTEP (*Flash-based SSDs*). Fuente: https://ostep.org

## 44.1 El medio: flash NAND y sus rarezas

La memoria flash guarda bits en celdas (SLC: 1 bit; MLC: 2; TLC: 3; QLC: 4 — más densidad, menos velocidad y resistencia), organizadas en una jerarquía con la asimetría que lo condiciona todo:

- **Página** (~4–16 KiB): unidad de **lectura** y de **programación** (escritura).
- **Bloque** (~1–8 MiB, cientos de páginas): unidad de **borrado**.

Las tres operaciones: leer página (~25–100 µs, uniforme — sin mecánica, sin seeks); **programar** página (~200–900 µs) — pero solo páginas *previamente borradas*; **borrar** bloque (~ms) — pone el bloque entero a 1s, destruyendo su contenido.

Consecuencia: **no hay sobrescritura in situ**. Modificar 4 KiB "en su sitio" exigiría leer el bloque entero, borrarlo y reescribirlo → carísimo y peligroso. Además el flash se **desgasta**: cada ciclo programa/borra daña la celda (~100k ciclos SLC, ~10k MLC, menos TLC/QLC).

## 44.2 La FTL: un LFS dentro del dispositivo

La SSD expone la interfaz clásica de bloques lógicos (para ser un "disco") mediante la **FTL** (Flash Translation Layer), que traduce direcciones lógicas → físicas. La FTL buena es **log-structured** (¡cap. 43 hecho hardware!):

- Las escrituras lógicas se **anexan** a páginas libres del bloque activo; la tabla de mapeo apunta la nueva posición; la página vieja queda **inválida**.
- **Garbage collection**: bloques con muchas páginas inválidas → copiar las vivas a otro sitio y borrar el bloque. El GC es el coste oculto: cada byte que el GC recoloca es **write amplification** (WA = escrito físico / escrito lógico).
- **Wear leveling**: repartir el desgaste — incluso mover datos fríos que "bloquean" bloques poco usados, para que todos los bloques envejezcan parejo.
- **Mapeo híbrido** (por bloques + por páginas para lo activo) o mapeo por páginas con la tabla parcialmente en RAM y el resto en flash: la tabla completa a nivel de página sería gigante (1 GB por TB, aprox).

## 44.3 Rendimiento y coste

Frente al HDD: lectura aleatoria ~1000× mejor (sin mecánica); secuencial también gana pero la brecha es menor; la escritura aleatoria *sostenida* depende del GC (una SSD llena y fragmentada se degrada — los fabricantes reservan **overprovisioning** para darle aire al GC). El compromiso perenne: $/GB del HDD sigue siendo mejor → jerarquías (SSD para caliente, HDD para frío).

Reglas prácticas derivadas:

- **TRIM/discard**: el FS avisa a la FTL de bloques liberados → menos páginas "vivas" falsas, GC más barato.
- El patrón de escritura importa: secuencial y por grandes trozos = WA ~1; aleatorio pequeño = WA alta y desgaste.

## Ampliación (no está en OSTEP)

- NVMe (colas paralelas, cap. 36) liberó el rendimiento que SATA estrangulaba; los SSD actuales: ~500k–1M IOPS, latencias ~10–100 µs.
- La FTL se está "abriendo": **ZNS** (Zoned Namespaces: zonas append-only expuestas al host — el FS hace de FTL, f2fs/btrfs las soportan) y **FDP**; en el extremo, open-channel SSDs. Motivo: dos logs apilados (LSM de RocksDB sobre FTL) multiplican la WA; alinear las capas la elimina — la lección de capas de todo el libro.
- **Optane/3D XPoint** (memoria persistente byte-direccionable) fue el intento de saltarse esta jerarquía; cancelado en 2022 — el flash + DRAM ganó por economía. Su nicho renace como CXL memory.
- Cifras de resistencia actuales: TLC ~1–3k ciclos con corrección de errores agresiva (LDPC) — el firmware moderno es mitad máquina de estados, mitad teoría de códigos (cap. 45).

## Referencias

- OSTEP cap. 44.
- Agrawal et al., *Design Tradeoffs for SSD Performance*, USENIX ATC 2008.
- Gupta, Kim & Urgaonkar, *DFTL: A Flash Translation Layer Employing Demand-based Selective Caching*, ASPLOS 2009.
- Bjørling et al., *ZNS: Avoiding the Block Interface Tax for Flash-based SSDs*, USENIX ATC 2021.
