# Análisis de código — GridCraft

Carpeta de documentación de análisis técnico del código fuente. Cada documento analiza un
aspecto del proyecto y se numera en orden de creación. Los documentos aquí son instantáneas
del estado del repositorio: cada uno indica sobre qué commit se hizo el análisis.

## Documentos

| #  | Documento                                | Alcance                                                                           | Commit base | Fecha      |
|--- |------------------------------------------|-----------------------------------------------------------------------------------|-------------|------------|
| 01 | [Análisis general del código](01-analisis-general-del-codigo.md) | Workspace completo: arquitectura, capas, motor de comandos, cálculo, testing, calidad | `3926d21`   | 2026-10-11 |

## Convenciones

- Idioma: español (los identificadores de código, crates y comandos se citan en su forma original).
- Métricas de líneas: contadas con `find <crate> -name '*.rs' | wc` (incluye tests); para la
  cifra canónica usar `cargo xtask stats`.
- Cada hallazgo cita `ruta/archivo.rs` (y línea cuando aplica) para poder verificarlo.
