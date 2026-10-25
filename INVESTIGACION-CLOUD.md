# Investigación: qué es algoritmo y qué es trabajo humano en Curator (v0.6.0)

Base: tag `v0.6.0` (commit `ac44bb2`), rama `investigacion/cloud-20261025`. Solo lectura: no se modificó
código ni dependencias. Etiquetas: **[IMPL]** = implementación leída, **[TEST EJEC]** = test
ejecutado, **[TEST LEÍDO]** = test solo leído, **[HIP]** = hipótesis.

Rutas abreviadas: `SM` = `packages/score_engine/lib/src/sync_map.dart`, `MT` = `.../midi_tempo.dart`,
`SF` = `.../score_form.dart`, `SC` = `.../curated_scene.dart`, `SR` = `.../scroll_map.dart`.

## A) Sincronización

**Qué es un tap.** Cada tap crea un `SyncAnchor(quarter, seconds)`: una negra de la partitura fijada a un
instante del audio (`SM:9-21`). El primer tap de la pieza es siempre la negra 0 (`SM:274-275`). [IMPL]

**Predicción de compás/pulso.** Con ≥2 anclas, el tempo es el del último segmento tapeado; antes, el
`defaultTempo` (`SM:278-280`). Se predice `b.departure + Δt·tempo`, se redondea al punto de rejilla más
cercano (compás o pulso, `SyncGrid`, `SM:45`, `SM:235-238`) y, si no avanza, se fuerza al siguiente
(`SM:281-283`). El ancla anterior queda re-temporizada porque el tempo entre anclas se deriva de ellas
(`SM:165-166`). Un tap a ≤0,15 s de un *warp* lo re-temporiza en vez de crear ancla (`SM:257-262`).
[IMPL] [TEST EJECUTADO `packages/score_engine/test/sync_map_test.dart:16-29`]

**Tempo inicial de MusicXML.** Es el **primer** `<sound tempo>` válido del archivo
(`score_metadata.dart:208-212`); si no hay, la app usa **100** (`app/lib/editor_controller.dart:299`).
Cambios de tempo posteriores escritos en el MusicXML no se leen para la sincronía. [IMPL]

**Ajuste a ataques (~80 ms).** Confirmado: `nearestOnset(t, 0.08)` en `app/lib/editor/sync_editing.dart:178`,
activado por defecto (`:55`). El detector es un flujo de energía logarítmica en tramas de 10 ms con
umbral local ±250 ms y separación mínima de 60 ms (`app/lib/audio_track.dart:70-97`). Es una corrección
del instante del tap, **no** alineación nota a nota: no sabe qué nota es ni consulta la partitura. [IMPL]
El test de onsets/tapeo lo reproduce a mano en vez de pasar por `tap()` (`app/test/waveform_test.dart:56`
«what the app does»). [TEST LEÍDO]

**MIDI de tempo.** Solo se leen eventos *Set Tempo* y el fin de pista; notas y compases se ignoran
(`MT:16-20`, `MT:47-64`). Requisitos: SMF (o RIFF `.rmi`) con división en ticks/negra, no SMPTE
(`MT:83-87`); sin tempo se asume 120 (`MT:42`). La negra 0 del archivo = inicio de la partitura (`MT:18`).
Se convierte en anclas: una por cambio de tempo, un warp por salto y una al final (`MT:242-265`).
Los warps se toman de los de la sincronía, o de la forma de la partitura **solo si la duración del MIDI se
parece más a la versión con repeticiones** (`editor_controller.dart:1153-1178`). [IMPL]
[TEST EJECUTADO `packages/score_engine/test/midi_tempo_test.dart:34-73`]

**Dónde se rompe el tempo constante.** Entre dos anclas el tempo es constante por diseño (`SM:50-52`,
interpolación lineal `SM:177-179`, `SM:187-189`); antes de la primera y después de la última continúa el
tempo vecino (`SM:175-176`). Un rubato dentro de un segmento queda aplanado: el músico debe añadir anclas.
[IMPL] [TEST EJECUTADO `sync_map_test.dart:31-39`]

**Posición escrita vs. interpretada.** La sincronía trabaja sobre la posición *interpretada* (negras tocadas)
y la traduce a la escrita por *pasadas* separadas por warps (`SM:55-60`, `SM:110-129`, `SM:204-214`).
En tapeo los warps los marca **el usuario** (`setJump`, `SM:331`; `sync_editing.dart:147`). La forma escrita
(repeticiones, casillas, D.C., D.S., coda, Fine) se calcula en `SF:160-163` y `SF:272-286`, pero solo se usa
para MIDI (`editor_controller.dart:1167-1170`). D.C./D.S./To Coda/Fine solo se reconocen por atributos de
`<sound>` (`SF:78-85`); un «D.S. al Coda» escrito solo como texto no cuenta. [IMPL]

## B) Dibujo por tiempo

Cadena: `t` → `SyncMap.quarterAtSeconds` / `secondsAtQuarter` por pasada → `ScrollMap.xAt(t)` (curva por
pasada con anclas en pulsos y/o en ataques **grabados en la partitura**, no del audio; `SR:180-196`,
`SR:207-283`, `SR:328`) → `SpacingPlan.topsAt(t)` (alturas de pentagramas, `SC:897`) → `isShownAt`
por regiones en negras convertidas a tiempo por pasada (`curation.dart:116-135`, `:345`). [IMPL]

- **Una vez por cambio de sincronía:** `setTimeline` reconstruye `ScrollMap` (`SC:319-322`); las curvas se
  resuelven al primer uso (`SR:270-272`).
- **Cacheado por clave:** el `SpacingPlan` se rehace solo si cambia la clave (curación, tamaño,
  `_timelineVersion`, etc.; `SC:597-598`, `SC:638`); la lista de visualización al cambiar tamaño/estilo
  (`SC:296-311`); teselas de pentagrama en LRU (`score_renderer.dart:412-416`).
- **Por fotograma:** `layoutAt`, `xAt`, fundido final y composición de teselas (`SC:683-700`). [IMPL]

**¿Mismo instante = mismo dibujo?** La exportación copia curación y sincronía en una escena propia
(`app/lib/video_export.dart:433-447`). `export_parity_test.dart:63-89` compara píxeles de vista previa vs.
`paintPreview` de la exportación en 6 instantes, incluido uno justo antes de un warp. [TEST LEÍDO]
No cubre el vídeo codificado por ffmpeg. Que *buscar* (seek) dé el mismo fotograma que reproducir:
no encontré estado entre fotogramas en `paint` [HIP], sin test que lo verifique. No hay mediciones de
rendimiento en el repo; no invento ninguna.

## C) Material propio

- **`.ccs`** = zip con `project.json` (todas las ediciones), la partitura **byte a byte**, el audio, imágenes
  y fuentes (`app/lib/project_file.dart:13-24`). La partitura no se reescribe. [IMPL]
- **Cambiar el MusicXML (Replace Score):** instrumentos se emparejan por id/nombre; textos por posición y
  palabras; posiciones (anclas, regiones, subtítulos) por **número de compás + distancia dentro**
  (`app/lib/score_replacement.dart:8-15`, `:101-109`). Anclas: conservan su segundo y se re-apuntan
  (`:149-153`); warps de MIDI se conservan hasta el primero que no cabe (`:155-170`). Se pierde lo que cae
  tras el último compás nuevo. [IMPL] [TEST LEÍDO `app/test/replace_score_test.dart:45-79`]
- **Garantía vs. ejemplo:** el código garantiza el traslado *por compás*, no por contenido musical. Si se
  inserta o elimina un compás, todo lo posterior queda apuntando al compás equivocado sin aviso específico
  [HIP derivada de `:101-109`]: el músico debe revisar anclas y warps. El test usa un caso donde coinciden.
- **Reutilizable sin rehacer:** sincronía (si la numeración de compases no cambia), curación por regiones,
  nombres, textos editados. **Revisa el músico:** compases añadidos/quitados, cambios de forma
  (repeticiones), y cualquier cambio de tempo escrito (no se relee, ver A).

## Tres hallazgos firmes

1. **La sincronía es semimanual.** El algoritmo predice compás/pulso desde el tempo previo y corrige el tap
   a un ataque en ±80 ms; el *qué* y el *dónde salta* lo decide el músico (taps, warps, corrección ↑/↓).
   Nada alinea notas del audio con notas de la partitura. [IMPL]
2. **Tempo constante a trozos.** Todo rubato entre dos anclas se aplana; la resolución expresiva depende de
   la densidad de anclas que el humano ponga (o del mapa MIDI). [IMPL] [TEST EJECUTADO]
3. **La forma escrita no guía el tapeo.** Repeticiones/casillas/D.C./D.S. se calculan solo para el MIDI;
   en tapeo los warps son manuales; además D.C./D.S. dependen de atributos `<sound>`, y las pruebas de forma
   no ejercen D.C./D.S./coda: `segnoSymbol`, `codaSymbol` y `sound()` se declaran
   (`score_form_test.dart:26-28`) pero ningún test los usa. [IMPL] [TEST LEÍDO]

## Dos incógnitas

1. Comportamiento de `_tap` en una pasada que sobrescribe anclas existentes entre warps de varias
   repeticiones (`SM:264-287`): no hay test con más de una repetición y retapeo. [HIP]
2. Igualdad entre *seek* y reproducción continua, y entre `paintPreview` y el archivo final codificado. [HIP]

## D) Invariantes

| # | Invariante | Código | Prueba | Límite |
|---|---|---|---|---|
| 1 | Simple: sin anclas, `t = leadIn + q·60/tempo` | `SM:173`, `SM:183` | `sync_map_test.dart:9` (ejecutada, pasa) | Tempo = primer `<sound tempo>` o 100 |
| 2 | Primer tap = negra 0 | `SM:274-275` | `sync_map_test.dart:16` (ejecutada, pasa) | No admite empezar a mitad |
| 3 | Tap predicho se ajusta a la rejilla y siempre avanza | `SM:281-283` | `sync_map_test.dart:16` (ejecutada, pasa) | Error > ½ compás → compás equivocado |
| 4 | Ajuste a ataque ≤ 80 ms | `sync_editing.dart:178`, `audio_track.dart:46-53` | `waveform_test.dart:44` (solo leída; se salta sin decodificador) | Reverberación/legato sin ataque |
| 5 | Cambio de tempo: constante entre anclas, lineal | `SM:177-179` | `sync_map_test.dart:31` (ejecutada, pasa) | Rubato intra-segmento perdido |
| 6 | Repetición: el tempo sigue a través del warp; la posición salta | `SM:110-129`, `SM:204-214` | `sync_map_test.dart:48` (ejecutada, pasa) | Warps manuales en tapeo |
| 7 | MIDI → anclas exactas por cambio de tempo | `MT:242-265` | `midi_tempo_test.dart:34`, `:51` (ejecutada, pasa) | Solo Set Tempo; no SMPTE |
| 8 | Forma → warps (repetición + casillas) | `SF:272-286` | `score_form_test.dart:48` (solo leída) | D.C./D.S./coda sin test |
| 9 | Vista previa = exportación (píxeles) | `video_export.dart:433-447` | `export_parity_test.dart:16` (solo leída) | No cubre ffmpeg ni seek |
| 10 | Replace Score re-apunta por compás + offset | `score_replacement.dart:101-109` | `replace_score_test.dart:45` (solo leída) | Compás insertado desplaza todo |

## Siguiente contraste pequeño

Añadir (en otra rama, fuera de esta investigación) un test de `ScoreForm.playedOrder` con
`<sound dalsegno>` + `<sound tocoda>` + segno/coda, y otro con «D.S. al Coda» solo como `<words>`, para
confirmar que el segundo produce orden lineal.

## Toolchain y tiempo

Intento único: SÍ, dentro del tope (≈6 min incl. instalación). Flutter 3.47.8 stable clonado en el scratchpad (fuera del repo), Verovio `version-6.3.0` con el parche del repo vía `make` (en `third_party/`, ignorado por git), hook nativo compilado (`libscore_engine.so`). Ejecutado: `flutter test test/sync_map_test.dart test/midi_tempo_test.dart test/beat_grid_test.dart` en `packages/score_engine` → **7/7 pasan** (`All tests passed!`). No se ejecutaron `waveform_test`, `score_form_test`, `export_parity_test` ni `replace_score_test` (fuera del único intento); quedan como solo leídos.

Minutos usados: ≈8 de reloj del contenedor (`date`: 14:17:19 → ~14:25 UTC). Velocidad de reproducción: `setRelativePlaySpeed` sin corrección de tono
(`app/lib/audio_track.dart:201-205`, «lower pitch»): confirmado. [IMPL]

## Referencias

Upstream xy057/curator (GPLv3), tag `v0.6.0`. Archivos citados arriba con `ruta:línea` relativos a la raíz.
