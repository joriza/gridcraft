# Análisis de código — GridCraft

Carpeta de documentación de análisis técnico del código fuente. Cada documento analiza un
aspecto del proyecto y se numera en orden de creación. Los documentos aquí son instantáneas
del estado del repositorio: cada uno indica sobre qué commit se hizo el análisis.

## Documentos

| # | Documento | Alcance | Commit base | Fecha |
| --- | --- | --- | --- | --- |
| 01 | [Análisis general del código](01-analisis-general-del-codigo.md) | Workspace completo: arquitectura, capas, motor de comandos, cálculo, testing, calidad | `3926d21` | 2026-10-11 |
| 02 | [Cómo iniciar la aplicación](02-como-iniciar-la-aplicacion.md) | Guía verificada de arranque: desktop, CLI headless, snapshot offscreen, web, troubleshooting | `a8285fc` | 2026-10-12 |
| 03 | [Uso del CLI y su huella](03-uso-del-cli.md) | Flujos de uso verificados, disco/RAM medidos, portabilidad y exe autónomo (crt-static) | `5218368` | 2026-10-12 |

## Convenciones

- Idioma: español (los identificadores de código, crates y comandos se citan en su forma original).
- Métricas de líneas: contadas con `find <crate> -name '*.rs' | wc` (incluye tests); para la
  cifra canónica usar `cargo xtask stats`.
- Cada hallazgo cita `ruta/archivo.rs` (y línea cuando aplica) para poder verificarlo.
