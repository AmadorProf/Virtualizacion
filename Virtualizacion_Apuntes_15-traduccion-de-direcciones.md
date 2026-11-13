# 15 · Mecanismo: traducción de direcciones

> Apuntes propios sobre el capítulo 15 de OSTEP (*Mechanism: Address Translation*). Fuente: https://ostep.org

## 15.1 La idea general

La virtualización de memoria repite la receta de la de CPU (cap. 6): **eficiencia con hardware + control del SO**. Aquí la técnica se llama **traducción de direcciones por hardware**: en *cada* acceso a memoria (fetch de instrucción, load, store), el hardware convierte la dirección **virtual** que emitió el proceso en la **física** donde están realmente los datos. El SO no interviene en cada acceso (sería lentísimo): **configura** el hardware en los momentos clave y gestiona la memoria; el hardware traduce en caliente.

De nuevo la ilusión: el proceso cree que su memoria empieza en 0 y es suya; la realidad es que está en cualquier parte de la RAM, compartida con otros diez procesos.

## 15.2 El esquema más simple: base y límite (reubicación dinámica)

Supuestos simplificadores del capítulo: espacio de direcciones contiguo, pequeño y que cabe entero en RAM.

Hardware necesario — dos registros **por CPU**, dentro de la MMU:

- **base**: dónde empieza en RAM física el espacio del proceso actual.
- **bounds (límite)**: cuánto mide.

En cada acceso:

```
física = virtual + base        (si virtual < bounds)
excepción → el SO mata al proceso   (si virtual ≥ bounds)
```

Se llama **reubicación dinámica**: el proceso se puede colocar (¡y mover!) en cualquier sitio de la RAM con solo cambiar `base`. Contraste con la reubicación **estática** de antaño (reescribir las direcciones del ejecutable al cargarlo): insegura y sin posibilidad de mover después.

## 15.3 Reparto de responsabilidades

**Hardware (MMU)**: modo kernel/usuario; registros base/bounds; sumar y comprobar en cada acceso; instrucciones privilegiadas para cargar base/bounds y para registrar los manejadores de excepción; lanzar la excepción ante accesos fuera de rango.

**SO**:

- **Al crear un proceso**: buscarle hueco en RAM (con una **free list** de huecos — cap. 17).
- **Al terminar**: devolver su memoria a la free list.
- **En cada cambio de contexto**: guardar base/bounds del saliente en su PCB y cargar los del entrante — los registros son únicos por CPU, la traducción es *por proceso*.
- **Ante excepción**: normalmente, matar al proceso infractor.

Mientras el proceso corre, el SO no toca nada: ejecución directa limitada, versión memoria.

## 15.4 El problema que motiva todo lo demás

Base y límite es simple, rápido y protege bien. Pero exige que el espacio del proceso sea **contiguo en RAM y se reserve entero**, incluido el hueco muerto entre heap y pila → **fragmentación interna** masiva (espacio reservado pero no usado). Con espacios de direcciones grandes y dispersos, es insostenible. Primera generalización: trocear por segmentos lógicos (cap. 16); solución definitiva: trocear en páginas fijas (cap. 18).

## Ampliación (no está en OSTEP)

- Base/límite no es solo historia: los **segmentos de x86 en modo real** funcionaban de forma parecida, y la idea renace donde se quiere traducción barata — p. ej. las *sandbox* basadas en desplazamiento de SFI, o los límites de memoria de WebAssembly (base + bounds check por acceso, a veces con trucos de guard pages).
- El par "sumar base + comprobar límite en cada acceso" cuesta ~0 hoy: va integrado en el pipeline. La lección general — *poner la operación por-acceso en hardware y la política en software* — es la misma que TLB/page-walk (cap. 19) y que EPT en hipervisores (`../virtualizacion.md` §5.5).

## Referencias

- OSTEP cap. 15.
- Denning, *Virtual Memory*, ACM Computing Surveys 1970.
- Jacob & Mudge, *Virtual Memory: Issues of Implementation*, IEEE Computer 1998 — panorámica de mecanismos de traducción.
