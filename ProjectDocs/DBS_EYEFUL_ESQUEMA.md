# Esquema de `dbs/EYEFUL`

Este documento describe la organización lógica de la base EYEFUL observada el
2026-09-16. Es un mapa técnico, no un catálogo de participantes ni una fuente
de etiquetas clínicas. Las rutas, identificadores y datos biométricos deben
permanecer locales y fuera de Git.

## Vista general

```text
EYEFUL/
├── EYEFULDB/                         captura original y fuentes de cámara
│   ├── eyeful00/, eyeful01/, eyeful02/
│   │   ├── *.svo                      vídeo ZED original y reloj fuente
│   │   ├── *.txt, *.csv, *.bag        metadatos, registros y auxiliares
│   │   ├── test/                      pruebas técnicas; no cohorte clínica
│   │   └── Corruptos/                 entradas excluidas de análisis
│   └── ...
├── EYEFULDB-DIST-INSERT/             vídeo derivado por sesión/tarea/cámara
│   └── <sesión>/<TAB|SHO|FLO>/cam*/
│       ├── *-lefti.mp4               vista izquierda derivada
│       ├── *timestamp.txt            timestamps/exportación histórica
│       └── *POSE_34J_3D.csv          pose preextraída, si existe
├── EYEFULDB-DIST-SPLIT/              sesiones estructuradas y wearables crudos
│   └── <sesión>/
│       └── WEARABLES-RAW/
│           ├── OGO/                  plantillas/fuerza plantar
│           ├── E4/                   paquete Empatica original
│           └── ACTILIFE/             exportación ActiLife original
├── EYEFULDB-OGO/                     repositorio central de OpenGo/OGO
├── EYEFULDB-E4/                      repositorio central de Empatica/E4
├── EYEFULDB-ACTILIFE/                repositorio central de ActiLife
├── EYEFULDB-CALIB/                   calibraciones de sensores/cámaras
├── EYEFULDB-labeling/                anotaciones; usar solo con protocolo explícito
└── EYEFULDB-ELAN-labeling.DONOTUSE/  anotaciones marcadas para no usar
```

Las tareas aparecen con los códigos `TAB`, `SHO` y `FLO`. La cámara de
referencia que selecciona el descubridor actual es `cam1` cuando está presente;
en su ausencia se conserva la primera cámara disponible, por lo que la cámara
concreta debe quedar registrada en cualquier manifiesto experimental.

## Modalidades y jerarquía de autoridad

| Modalidad | Fuente primaria | Formatos habituales | Papel en el estudio |
|---|---|---|---|
| Vídeo ZED | `EYEFULDB/eyeful*/` | `.svo` (y potencialmente `.svo2`) | Imagen izquierda, índice fuente y timestamp hardware. Es el reloj maestro. |
| Vídeo derivado | `EYEFULDB-DIST-INSERT/` | `.mp4`, `.txt`, CSV de pose | Revisión visual o compatibilidad. No define por sí solo el tiempo de adquisición. |
| Pose | Junto al vídeo derivado o reextraída desde SVO | `*POSE_34J_3D.csv` | Señal cinemática; requiere comprobar orden y procedencia temporal. |
| OGO/OpenGo | `WEARABLES-RAW/OGO/` y `EYEFULDB-OGO/` | `.txt`, CSV | Fuerza plantar, CoP y señales inerciales/biomecánicas. |
| Empatica E4 | `WEARABLES-RAW/E4/` y `EYEFULDB-E4/` | `.zip`, CSV | ACC, EDA, BVP, temperatura, IBI y HR cuando estén disponibles. El paquete original conserva sus metadatos de reloj. |
| ActiLife | `WEARABLES-RAW/ACTILIFE/` y `EYEFULDB-ACTILIFE/` | `*-RAW.csv`, `AGD-1sec.csv` | Actividad, aceleración y contexto temporal/postural. |
| Calibración | `EYEFULDB-CALIB/`, `EYEFULDB-DIST-CALIB*`, `LEICA_calib/`, `cameras/` | archivos de cámara y auxiliares | Geometría, validación instrumental y configuraciones; no sustituye datos de ensayo. |
| Anotaciones | `EYEFULDB-labeling/` | formatos de etiquetado | Evaluación/ground truth solo bajo un protocolo separado de inferencia. |

## Relación lógica por ensayo

```text
subject_id + sesión + tarea + cámara
│
├── SVO/SVO2 original
│   ├── timestamp hardware por frame
│   └── lectura directa → pose canónica → biomarcadores de vídeo
│
├── vídeo derivado (opcional)
│   ├── MP4 / pose CSV preexistente
│   └── requiere mapa explícito SVO ↔ derivado si se usa temporalmente
│
└── sesión en DIST-SPLIT
    └── WEARABLES-RAW
        ├── OGO
        ├── Empatica E4
        └── ActiLife
             ↓
      sincronización por reloj original, offset documentado y control de calidad
```

La asociación válida es por ensayo, no por proximidad de nombre ni por la
duración aparente. El manifiesto del proyecto registra `subject_id`, `trial_id`,
disponibilidad y estado de cada modalidad; los paths sensibles se conservan
localmente.

## Colecciones derivadas, históricas o auxiliares

| Grupo | Carpetas | Uso recomendado |
|---|---|---|
| Derivados de inserción/corte | `EYEFULDB-DIST-INSERT-PRECUTTING/`, `EYEFULDB-DIST-EXTRA/`, `EYEFULDB-DIST-IMBALANCE/` | Inspección histórica o comparación técnica; no asumir equivalencia con el SVO actual. |
| Calibración derivada | `EYEFULDB-DIST-CALIB/`, `EYEFULDB-DIST-CALIB-ng/`, `EYEFULDB-DIST-CALIB-ng-depthNeural/` | Usar solo si se documenta la versión de cámara y calibración. |
| Históricos/deprecados | `EYEFULDB-DIST-DEPRECATED/`, `EYEFULDB-DIST-DEPRECATED-ng/`, `EYEFULDB-DIST-DEPRECATED-ng2/` | Evidencia de conversiones previas; no fuente canónica de tiempo. |
| Organización experimental | `EYEFULDB-DISTRACTOR/`, `EYEFULDB-REMNANTS/`, `pruebas/` | Revisar individualmente antes de incorporarlas a una cohorte. |
| Datos externos/auxiliares | `GREW-dataset/`, `T3.3 Recordings and data capture for the clinical evaluation/` | No mezclar con EYEFUL sin definir procedencia y protocolo. |
| Copias o exclusiones | `EYEFULDB-E4-BACKUP-WITH-USERS/`, `.Trash-*`, `EYEFULDB-ELAN-labeling.DONOTUSE/` | No usar como fuente de análisis. |

## Reglas operativas

1. **Tiempo:** SVO/SVO2 directo es la autoridad. Los MP4 pueden contener
   frames interpolados o insertados y nunca se temporizan con `frame / fps`.
2. **Integridad:** exportar y auditar la línea temporal SVO antes de fusionar
   sensores. Bloquear características temporales si hay huecos, jitter elevado,
   índices no contiguos o posiciones fuente no legibles.
3. **Sensores:** conservar el reloj original, zona horaria, offset, frecuencia
   nominal, cobertura y calidad de OGO, E4 y ActiLife por separado.
4. **Cohorte:** dividir entrenamiento, validación y prueba por `subject_id`.
   Una tarea, cámara o sensor de un mismo sujeto no puede cruzar particiones.
5. **Etiquetas:** `ground_truth` y anotaciones son opt-in y pertenecen a la
   evaluación; no deben alimentar el pipeline de inferencia.
6. **Privacidad:** no copiar vídeo, señales, IDs, rutas de sesión ni resultados
   clínicos al repositorio, fixtures, logs públicos o artefactos versionados.

## Comandos del proyecto relacionados

```bash
# Descubrimiento no destructivo de rutas y disponibilidad.
python3 scripts/discover_eyeful_manifest.py ...

# Auditoría SVO directa, limitada explícitamente; no abre sensores.
PYTHONPATH=src ./.venv/bin/python scripts/audit_eyeful_svo_sources.py \
  --manifest_csv resultados/eyeful_manifest_raw.csv \
  --output_csv resultados/eyeful_svo_source_qc_001.csv \
  --require_raw_multimodal --offset 0 --limit 1
```

El segundo comando solo habilita la siguiente fase si el estado temporal es
`pass`; después se debe comprobar solape y sincronización de cada wearable.
