# 50 · AFS: el Andrew File System

> Apuntes propios sobre el capítulo 50 de OSTEP (*The Andrew File System (AFS)*). Fuente: https://ostep.org

## 50.1 Otro objetivo, otro diseño

AFS (Carnegie Mellon, años 80) atacó lo que NFS descuidó: la **escala** — ¿cuántos clientes puede servir un servidor? Cada decisión de diseño sale de medir dónde gasta el tiempo el servidor y quitárselo de encima. También ofreció algo que NFS no tenía: **semántica de consistencia simple y explicable**.

## 50.2 AFSv1 y la medición

Diseño inicial: los clientes cachean **ficheros enteros** en su **disco local** (no páginas en RAM como NFS): al abrir, el fichero completo viaja del servidor al disco del cliente; reads/writes son locales; al cerrar (si hubo escritura), vuelve entero al servidor.

La medición de AFSv1 encontró dos cuellos: el tráfico de **rutas completas** (el servidor recorría `/home/ana/doc.txt` componente a componente en cada petición) y el exceso de mensajes **TestAuth** (los clientes preguntando "¿sigue válida mi copia?" — el mismo diluvio de GETATTRs de NFS). El servidor gastaba su CPU en eso, no en servir datos.

## 50.3 AFSv2: callbacks y FIDs

Las dos correcciones que definieron el sistema:

- **FIDs** (como los handles de NFS): el cliente recorre las rutas pieza a pieza *una vez* y cachea directorios → el servidor deja de recorrer rutas.
- **Callbacks**: en vez de que el cliente pregunte, **el servidor promete avisar** ("te llamaré si alguien modifica este fichero"). Mientras no llegue aviso, la copia local se usa sin verificar — polling → interrupciones (el mismo salto conceptual del cap. 36). El precio: el servidor ahora tiene **estado** (recuerda qué cliente cachea qué).

## 50.4 La semántica: consistencia a la última cerradura

Con callbacks, AFS ofrece una regla limpia, explicable en una frase: **los cambios se publican al cerrar el fichero, y el servidor invalida entonces todas las copias cacheadas** (*update visibility on close* + callback breaks). El siguiente `open` en cualquier cliente trae la versión nueva. Entre concurrentes escribiendo a la vez, gana el **último close** (last-writer-wins del fichero completo — nada de mezclas por bloques como podía pasar en NFS).

Comparación de fondo con NFS: la ventana de ~3 s de inconsistencia de NFS desaparece; a cambio, el estado del servidor complica la **recuperación** — si el servidor rearranca, pierde la lista de callbacks y todos los clientes deben revalidar todo (y si un *cliente* estuvo desconectado, pudo perderse un aviso). El péndulo stateless/stateful, con sus costes a la vista en ambos extremos.

## 50.5 Rendimiento: cuándo gana cada uno

El análisis comparativo del capítulo deja el mapa: AFS brilla en **relecturas** (disco local, ni red ni servidor) y en cargas de fichero completo; NFS gana en **lecturas/escrituras parciales de ficheros grandes** (AFS mueve el fichero entero aunque toques un byte — el caso patológico: sobrescribir 40 bytes de un fichero de GBs). Para el uso común (ficheros pequeños, releídos), empatan o gana AFS con mucho menos servidor.

Herencia extra de AFS: espacio de nombres **global** único (/afs/...), seguridad seria (Kerberos, ACLs por directorio), y herramientas de administración de verdad — el "producto completo" del que NFS carecía.

## Ampliación (no está en OSTEP)

- AFS como sistema quedó en nichos (OpenAFS sobrevive en algunas universidades), pero sus ideas ganaron por otros caminos: los callbacks son las **delegaciones/leases de NFSv4** y los **oplocks de SMB**; el caching agresivo de fichero completo es **Dropbox/OneDrive/Google Drive** (sincronización + notificaciones = AFS a escala de internet); y **Coda** (su sucesor en CMU) exploró la operación desconectada que hoy damos por sentada en móviles.
- La metodología del capítulo es la mitad de su valor: *medir el sistema v1, encontrar los cuellos reales, rediseñar el protocolo alrededor de ellos* — el patrón LFS (cap. 43) y el patrón de toda buena ingeniería de sistemas.

## Referencias

- OSTEP cap. 50.
- Howard et al., *Scale and Performance in a Distributed File System*, ACM TOCS 1988 — el paper de AFSv2 y su metodología de medición.
- Satyanarayanan, *The Evolution of Coda*, ACM TOCS 2002.
- RFC 7530 (NFSv4, para ver los callbacks renacidos como delegaciones).
