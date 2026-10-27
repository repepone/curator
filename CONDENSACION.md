# La condensación en Curator, explicada para músicos

Base leída: `main` en `1815166` (v0.6.0 + 24 commits). En `v0.6.0` el parche de Verovio pesa
6,9 KB; el de 71 KB que pediste mirar recién aparece en `main`. Trabajo de solo lectura: no se tocó
código. Etiquetas: **[LEÍDO]** = lo dice el código citado; **[HIPÓTESIS]** = deducción mía, sin
prueba en el repo.

Abreviaturas de rutas: `C` = `packages/score_engine/lib/src/condensing.dart`,
`CU` = `.../curation.dart`, `CS` = `.../curated_scene.dart`, `SP` = `.../spacing_plan.dart`,
`AC` = `.../auto_curate.dart`, `SM` = `.../score_metadata.dart`, `P` = `patches/verovio-curated-score.patch`.

## Lo primero: en Curator hay dos "pliegues" distintos

1. **Ocultar instrumentos cuando no tocan** (lo que en la app es "curación"): cada instrumento tiene
   una pista con regiones; fuera de ellas su pentagrama desaparece y los demás se acomodan.
   [LEÍDO `SP:14-28`]
2. **Condensar dos músicos en una línea** (Flauta 1 + Flauta 2 → "Fl. 1.2"), al estilo Dorico.
   [LEÍDO `C:6-12`]

Lo que el código llama *condensing* es solo el punto 2. El punto 1 es la curación. Abajo hablo de los dos.

## 1. ¿Qué se pliega y con qué granularidad?

- **Unidad que se oculta: el instrumento (la "parte")**, no el sistema ni la familia. Un piano con
  dos pentagramas se mueve como un bloque. [LEÍDO `SP:20-21`]
- **Unidad que se condensa: exactamente un par de músicos.** No hay tríos ni familias enteras: el tipo
  `CondensedGroup` guarda dos ids. [LEÍDO `C:20-21`, `C:51-60`]
- **Decisión de mostrar u ocultar: por región**, y las regiones están en negras de la partitura, así
  que pueden empezar en cualquier punto. Las que arma la app sola caen en barras de compás.
  [LEÍDO `CU:237-245`]
- **Decisión de cómo escribir la línea condensada: por compás.** Cada compás se clasifica por separado.
  [LEÍDO `C:327-347`]
- **Cuándo se ve la línea condensada:** cuando el par está activado y *los dos* instrumentos están
  visibles en ese momento; si solo uno está visible, se ve su pentagrama propio. [LEÍDO `CS:617-624`]
  En el cambio, una línea se desvanece mientras la otra aparece. [LEÍDO `CS:626-637`]
  Ojo: mientras el par está condensado, la app le da **una sola pista** a los dos (la del primero, y
  la del segundo copia sus regiones). [LEÍDO `app/lib/editor_controller.dart:1011-1013`] En la práctica,
  entonces, se ve la línea "1.2" en todo el tramo, y los solos aparecen como "1."/"2." dentro de ella.
  [HIPÓTESIS derivada de esas líneas]
- Por defecto ningún par está activado: lo prende el usuario. [LEÍDO valor por defecto vacío en
  `app/lib/project_state.dart:41`, aplicado al abrir en `app/lib/editor_controller.dart:296`]

## 2. Criterios para decidir que un pentagrama "no hace falta"

Reglas concretas que encontré:

1. **"Toca" = tiene al menos una nota que no sea silencio en ese compás.** [LEÍDO `SM:177-180`]
2. **Ocultar automático:** se muestra cada tramo de compases donde toca, y se "puentea" un silencio
   de hasta **1 compás** entre dos tramos para que no parpadee. [LEÍDO `AC:1-26`] El número 1 es una
   decisión del autor, no una regla de orquestación. [HIPÓTESIS sobre la intención; el valor sí está
   LEÍDO en `AC:10`]
3. **Ordenar a mano:** el menú "Tidy" puentea silencios de N compases o borra regiones cortas.
   [LEÍDO `app/lib/instruments_toolbar.dart:104-140`]
4. **Pentagramas que nunca suenan** dentro de un instrumento de varios (p. ej. la mano izquierda
   vacía) se sacan; un instrumento de un solo pentagrama siempre se conserva, aunque calle.
   [LEÍDO `SM:197-202`]

Lo que **no** hay (lo busqué y no está): detección de unísonos o duplicaciones *entre instrumentos
distintos que no forman un par*, jerarquía orquestal (melodía antes que relleno), pesos por familia,
análisis armónico. [LEÍDO por ausencia en `C`, `CU`, `AC`; ninguna regla así]

Para **condensar**, las reglas de quién puede juntarse con quién son:

- **Pares automáticos:** dos partes **contiguas** con el mismo nombre y número distinto ("Trompa en
  Fa 1"/"2"), de un pentagrama, una sola voz, y las mismas claves, armaduras, transposición y compases
  todo el tiempo. De a dos: 1.2 y 3.4. [LEÍDO `C:131-145`, `C:165-173`, `C:213-216`]
- **Pares a mano:** cualquier par de un pentagrama con iguales claves, transposición y compases
  (flauta + violín sí; flauta + clarinete en Si♭ no). Percusión sin altura solo con percusión.
  [LEÍDO `C:84-88`; diálogo en `app/lib/condensing_editor.dart:8-29`]
- Violín I / II **no** se condensan solos porque no son "músicos numerados". [LEÍDO `C:135-136`]
- Que el nombre termine en número es una heurística del autor (expresión regular). [LEÍDO `C:35-42`]

## 3. Qué pasa con las notas al condensar

Sí, **se reescriben las notas**: Curator agrega a la partitura un pentagrama nuevo (`cond-P2-P3`) antes
de grabarla, y Verovio lo graba como otro más. [LEÍDO `C:10-12`, `C:179-208`] Cada compás cae en uno
de seis casos [LEÍDO `C:112-130`, `C:386-398`]:

| Caso | Qué muestra |
|---|---|
| Los dos callan | los silencios del primero |
| Toca uno solo | sus notas, con **"1."** o **"2."** donde empieza |
| Mismas notas | las del primero, con **"a2"** |
| Mismo ritmo y el 2.º nunca por encima | **acordes** en una voz (Verovio elige las plicas) |
| Cualquier otra cosa | **dos voces**, plicas del 1.º arriba y del 2.º abajo |

- El rótulo solo se repite cuando cambia quién toca. [LEÍDO `C:334-346`] Si no son "músicos numerados",
  el rótulo es la abreviatura ("Fl.", "Ob.") en vez de "1."/"2.". [LEÍDO `C:197-198`]
- En acordes, las notas del 2.º se agregan a las del 1.º sin repetir alturas, conservando ligaduras de
  prolongación. Pierden plica, barra, letra y dinámica propia. [LEÍDO `C:496-520`]
- Las dinámicas del 2.º van **debajo** y las del 1.º, **arriba**. [LEÍDO `C:454-466`]
- Si el 2.º está escrito en otra armadura, se re-escriben las alteraciones contra la del 1.º.
  [LEÍDO `C:140-142`, `C:567`]
- Ligaduras o reguladores que quedan con una punta sola se borran. [LEÍDO `C:349`, `C:352-358`]
- **No se cambia la cabeza de nota** para marcar el unísono. [LEÍDO por ausencia en `C:403-451`]

## 4. La guía

**No hay un pentagrama "guía" que nunca se pliegue.** No encontré ningún concepto de
pentagrama fijo, de referencia o "siempre visible" en curación ni en la escena. [LEÍDO por ausencia;
búsqueda de *pinned/always/guide* sin resultados pertinentes] Lo único "fijo" es que un instrumento de
un solo pentagrama nunca se borra del grabado (`SM:197-198`). Pero eso no impide que se oculte en pantalla.
Si querés una guía, hoy hay que dejarle a ese instrumento una región que cubra toda la obra.
[HIPÓTESIS de uso, coherente con `CU:237-245`]

## 5. Cómo llega a Verovio, y qué es el parche de 71 KB

**Lo que se le pide a Verovio** es poco: un MusicXML ya condensado y un solo sistema infinito, sin
saltos de página (`"breaks": "none"`). [LEÍDO `packages/score_engine/src/verovio_bridge.cpp:666-680`;
condensado antes de grabar en `.../score_text.dart:140-143`] Después, un "dispositivo grabador" propio
anota cada trazo que Verovio dibuja. [LEÍDO `verovio_bridge.cpp:85-112`] **Ocultar y mover pentagramas
lo hace Curator en su propio dibujo, no Verovio.** [LEÍDO `SP:14-28`]

**El parche (en una carilla, sin C++).** Lo importante: **no contiene nada de condensación.**
[LEÍDO, todo `P`] Hace cuatro cosas:

1. **Tamaños tipográficos por tipo de símbolo** (la mayor parte de los 71 KB): unas 25 perillas nuevas
   (dinámicas, articulaciones, alteraciones, claves, silencios, cabezas, calderones…), cada una un
   porcentaje del tamaño del pentagrama, más estilo y espaciado de digitaciones y cifrados.
   [LEÍDO `P:14-80`, `P:351-477`, y las líneas de dibujo `P:478-1260`]
2. **Arreglos de lectura del MusicXML:** respeta tamaño y espaciado de textos que pone la app, conserva
   los ids de textos (para editarlos después) y, sobre todo, hace que **un cambio de armadura a mitad
   de obra llegue a cada instrumento con su propia armadura**: sin esto, los transpositores copian la
   del primero. [LEÍDO `P:225-260`, `P:265-312`, `P:313-347`]
3. **Arreglos de dibujo:** que las alteraciones no choquen con articulaciones agrandadas, que un cambio
   de armadura no "cancele" mal, y que respiraciones y cesuras queden entre las notas vecinas.
   [LEÍDO `P:121-140`, `P:205-216`, `P:572-610`]
4. **Un registro de avisos por hilo**, para grabar varias partituras a la vez. [LEÍDO `P:91-101`, `P:1270-1273`]

## 6. ¿Qué se puede hacer en el navegador con verovio-toolkit.js (WASM) sin recompilar?

- **Condensar pares (reescribir el MusicXML):** sí. Es texto XML transformado *antes* de Verovio; se
  puede portar a JavaScript. [LEÍDO que es pre-proceso: `score_text.dart:140-143`; la portabilidad es
  HIPÓTESIS]
- **Ocultar por tramos de compases (curación automática):** la regla es trivial (`AC:10-26`) y se puede
  portar. [HIPÓTESIS]
- **Mostrar/ocultar pentagramas según el tiempo:** Curator lo hace con su propio dibujador, que no
  existe en el toolkit web. En web habría que ocultar grupos del SVG de Verovio o regrabar con otra
  lista de pentagramas. [HIPÓTESIS] Verovio estándar trae una opción para condensar sistemas vacíos,
  pero funciona por sistema, no por momento de reproducción. [HIPÓTESIS: no está en este repo]
- **Exige tocar el motor:** las perillas tipográficas del parche y los arreglos de armaduras para
  instrumentos transpositores. Sin el parche, un par condensado de trompas con cambio de armadura a
  mitad de obra podría mostrar la armadura equivocada. [HIPÓTESIS derivada de `P:265-267`]

## 7. Costo y riesgo: ¿cuánto depende de Flutter?

- `condensing.dart` solo usa el paquete XML y una utilidad de comparación de listas de Flutter: lógica
  pura y portable. [LEÍDO imports de `C:1-4`] `auto_curate.dart` no importa nada. [LEÍDO]
- `curation.dart` usa utilidades de Flutter pero no dibuja. `spacing_plan.dart` no importa Flutter.
  [LEÍDO imports]
- `curated_scene.dart` (qué pentagrama se ve, fundidos, dibujo) depende de `dart:ui` de Flutter y del
  dibujador propio: **es la parte que no se lleva tal cual**. [LEÍDO imports de `CS`]
- **Pruebas:** la condensación tiene 2 pruebas (los seis casos de compás y los pares de la demo).
  [LEÍDO `packages/score_engine/test/condensing_test.dart:126`, `:173`] No ejecuté nada esta vez.
- **Riesgo:** el motor (Verovio + parche) hace el grabado. La *decisión musical* (qué se condensa,
  cuándo se oculta) vive en Dart y en la app. [LEÍDO]

## Qué haría yo en VEM

Orden mínimo para plegar por familias en un visor web con Verovio, con guía (todo **[HIPÓTESIS]**):

1. Leer el MusicXML y armar, por instrumento, la lista "toca / no toca" por compás (regla de `SM:177-180`).
2. Asignar familias por nombre o `instrument-sound` y una pista por familia (unión de sus instrumentos).
3. Generar tramos visibles con puente de 1–2 compases (regla de `AC`), por familia.
4. Marcar un instrumento como **guía**: tramo único de toda la obra, nunca se oculta.
5. Grabar con verovio-toolkit.js en un sistema único (`breaks: none`), una sola vez.
6. Mostrar u ocultar por tiempo los grupos SVG de cada pentagrama y reacomodar las alturas, sin regrabar.
7. Recién después, condensar pares 1/2 portando las seis reglas de `C:386-398` como pre-proceso del XML.
8. Probar con una obra con trompas transpositoras y cambio de armadura, para medir si hace falta el parche.
9. Dejar la tipografía fina (perillas del parche) para el final o para un Verovio recompilado.
10. Tests: un caso por cada regla de compás y uno de familia que entra y sale.

## Referencias

Upstream xy057/curator (GPLv3), rama `main` en `1815166`. Rutas relativas a la raíz del repo.
