# 01 · Análisis general del código — GridCraft

| | |
| --- | --- |
| **Commit base** | `3926d21` ("Fix-up: #423's table-name round-trip test…", 2026-10-11) |
| **Alcance** | Workspace completo de Rust: 12 crates, 3 apps, xtask |
| **Método** | Lectura de docs de alto nivel (`README.md`, `docs/architecture.md`), métricas medidas sobre el árbol de fuentes, y mapeo por crate con evidencia `path:line` |
| **Estado del proyecto** | Alpha (según `README.md`): ~94,5 % de paridad de comandos del ribbon, ~50 % "listo para trabajo real" ponderado |

---

## 1. Qué es GridCraft

Planilla de cálculo **clean-room** en Rust puro que apunta a paridad con Microsoft Excel (y a
superarlo en velocidad, apertura y control por agentes). Corre nativo en macOS, Windows, Linux y
BSD, y en la web vía WASM. No hay JS/TS escrito a mano: la app web es el mismo código Rust
compilado a `wasm32-unknown-unknown`.

Números clave del `README.md`:

- **274/290 comandos** del ribbon implementados (94,5 %) según `docs/parity-checklist.md`
  (generado por `cargo xtask parity` desde `crates/engine/src/catalog.rs`).
- **513 de 523 funciones** de hoja de cálculo (`docs/function-parity.md`).
- Formato nativo **XLSX** con round-trip de estilos, fórmulas (incluyendo arrays dinámicos),
  tablas, formato condicional, validación, comentarios, hipervínculos, gráficos, imágenes,
  sparklines, PivotTables y configuración de impresión. Además importa datos de `.xls` (97-2003),
  `.xlsb` y `.ods`, y lee/escribe CSV/TSV.

## 2. Métricas del workspace

Medición propia sobre `HEAD` (incluye archivos de tests; ver §11 para una discrepancia con
`docs/architecture.md`):

| Capa | Crate | Archivos `.rs` | Líneas |
| --- | --- | ---: | ---: |
| L0 | `core` | 5 | 1.408 |
| L0 | `numfmt` | 6 | 2.376 |
| L1 | `formula` | 9 | 2.375 |
| L1 | `functions` | 16 | 15.238 |
| L2 | `model` | 11 | 2.960 |
| L2 | `calc` | 5 | 4.226 |
| L2 | `xlsx` | 45 | 19.358 |
| L2 | `pdf` | 3 | 952 |
| L3 | `chart` | 8 | 3.860 |
| L5 | `engine` | 41 | 29.876 |
| L6 | `ui-egui` | 74 | 30.700 |
| L6 | `mcp` | 6 | 1.571 |
| L7 | `apps/gridcraft` (desktop) | 7 | 1.712 |
| L7 | `apps/gridcraft-cli` | 4 | 2.199 |
| L7 | `apps/gridcraft-web` | 2 | 404 |
| — | `xtask` | 8 | 1.571 |
| | **Total** | **250** | **~120.800** |

Otros indicadores:

- **~1.308 tests** (`#[test]`) en todo el workspace; 30+ archivos de test de UI con `kittest`
  en `crates/ui-egui/tests/`.
- **~300 entradas** en el catálogo de paridad (`catalog.rs`); el motor registra además
  primitivas fuera del catálogo (`cell.set`, `selection.set`, `range.setValues`, …); el
  `docs/architecture.md` habla de ~313 comandos totales.
- Edición 2024 de Rust, `rust-version = "1.90"`, `unsafe_code = "forbid"` a nivel workspace
  (`Cargo.toml`, `[workspace.lints.rust]`).

## 3. Arquitectura en capas

La estratificación está **forzada por herramienta**: `cargo xtask layers` rechaza dependencias
hacia arriba. Nada por debajo de L6 depende de `egui`/`eframe`/`winit`/`rfd`, de modo que la UI
es reemplazable sin tocar el motor.

```text
L0  core (direcciones, Value, CellError, fechas) · numfmt (códigos de formato numérico)
L1  formula (lexer, parser, AST, printer, adjust) · functions (biblioteca de funciones)
L2  model (workbook COW) · calc (evaluador + grafo de dependencias) · xlsx (E/S) · pdf
L3  chart (layout y rasterizado de gráficos, sin toolkit)
L5  engine (Session, historial, comandos, I/O)
L6  ui-egui (UI Excel-like + canal de control) · mcp (servidor MCP)
L7  apps: gridcraft (desktop), gridcraft-cli, gridcraft-web (WASM)
```

Observaciones de diseño destacables:

- **`pdf` es autónomo** (sin dependencias del engine): escritor PDF 1.7 con las 14 fuentes
  estándar, vectores e imágenes RGBA, usado para imprimir/exportar.
- **`chart` no usa toolkit**: resuelve series desde el workbook (`crates/chart/src/resolve.rs`),
  produce primitivas de dibujo (`render.rs`) y tiene `rasterize()` a RGBA para PNG; sirve igual
  para UI, PDF y tests.
- **`xlsx` trabaja solo con slices de bytes** (WASM-safe), con límites defensivos: 512 MB por
  parte OPC y 1 GB por paquete. Lector *leniente* que devuelve `(Workbook, ReadReport)` con
  advertencias en lugar de fallar.

## 4. El patrón rector: todo es un comando

El corazón del proyecto es que **toda acción visible es un comando ejecutable por cualquier
superficie** (UI, CLI, canal de control JSON, MCP).

- Definición: `CommandSpec { id, label, menu, shortcut, params, enabled, run, journal, undoable }`
  en `crates/engine/src/cmd/mod.rs:42-58`; la macro `cmd!` (línea ~90) construye los literales.
- Registro: cada submódulo de `crates/engine/src/cmd/` (`edit.rs`, `format.rs`, `file.rs`,
  `data.rs`, `insert.rs`, `pivot.rs`, …) exporta `pub fn specs() -> Vec<CommandSpec>`; se
  agregan en `command_specs()` dentro de un `OnceLock` estático (`cmd/mod.rs:139-162`).
- Ejecución única: `Session::execute(id, params)`. Errores tipados en `EngineError`
  (UnknownCommand, Disabled, BadParams, NoDocument, Other, Internal).
- **Regla never-crash**: `Session::execute` atrapa panics escapados y restaura el workbook.
  Programático nunca abre diálogos: con params vacíos corre con defaults o devuelve error; solo
  la invocación de menú abre diálogo vía `UiRequest::Dialog`.
- Catálogo de paridad: `crates/engine/src/catalog.rs` lista el árbol ribbon/menú de Excel
  (solo *feature names*, clean-room) y `cargo xtask parity` genera el checklist.

Ejemplos representativos: `cell.set` (`cmd/edit.rs:13`), `home.bold` (`cmd/format.rs:12`),
`file.new` con `noundo` (`cmd/file.rs:10`).

## 5. Motor de cálculo (`crates/calc`)

- **Evaluador**: `eval.rs` con `Evaluator<Host>` sobre el trait `Host` (inversión de
  dependencias: calc no conoce model directamente). Resolución de referencias en
  `reference_areas()` / `deref()`.
- **Recálculo incremental** (`recalc.rs`): grafo de dependencias con cierre transitivo de
  dependientes (BFS) y evaluación en orden topológico del subgrafo sucio. Los precedentes de
  rango ancho se indexan con **árboles de segmentos por columna** (`Intervals<T>`) para consultas
  O(log n + k); los de celda única van en `HashMap<CellRef, HashSet<Key>>`.
- **Arrays dinámicos y spills** con reporte de `#SPILL!`; **ciclos** → `#CIRC!`, salvo cálculo
  iterativo activado (`PassHost::iterate_loops`, techo `MAX_ITERATION_WORK = 10M` evaluaciones).
- Evaluación on-demand como red de seguridad (`Host::cell_value()` evalúa precedentes sucios al
  leer).
- **Gap conocido**: cálculo single-threaded (`docs/hardware-parity.md`); es uno de los frentes
  próximos según el roadmap.

## 6. Fórmulas y funciones

- **`formula`**: tokenizer (`lexer.rs`), parser descendente recursivo (`parser.rs`), AST
  (`ast.rs`: `Expr` con `BinOp`, `Reference`, `FunctionCall`, `Let`, `Lambda`, …), impresores
  A1 y R1C1 (`printer.rs`), ajuste de referencias para copiar/rellenar/edits estructurales
  (`adjust.rs`), y entrada por locale (`input.rs`; tests en `crates/formula/tests/spanish_input.rs`).
- **`functions`**: cada función se describe con `FnSpec` (nombre, aridad, categoría, args
  escalares, firma, descripción) y se implementa como función Rust pura sobre argumentos ya
  evaluados; `lookup()`/`call()` aplica el *array lifting* automático de Excel. Las formas
  especiales (ROW, OFFSET, INDIRECT, LET, LAMBDA, SUBTOTAL, AGGREGATE…) se delegan al host
  (calc/engine) porque necesitan semántica del evaluador.
- **i18n de fórmulas**: la UI ofrece nombres de función en español, coma decimal y `;` como
  separador (`docs/formula-input.md`); el almacenamiento y las APIs programáticas quedan en
  sintaxis canónica inglesa.

## 7. Modelo de datos y formatos de archivo

- **`model`**: `Workbook`/`Sheet` con **cell store copy-on-write** (bandas de 64 filas detrás de
  `Arc`), estilos internados (`StyleTable`/`StyleId`), nombres definidos, tablas, formato
  condicional, validación, comentarios, gráficos, shapes, imágenes, sparklines, print settings y
  protección (hash de contraseña en `password.rs`). El COW hace que los snapshots de undo sean
  casi gratis.
- **`xlsx`** (clean-room sobre ECMA-376/ISO 29500): lectura `read_xlsx(bytes)` en
  `crates/xlsx/src/read.rs:67` con contexto compartido (`Ctx`) y reporte de advertencias;
  escritura `write_xlsx(wb)` en `write.rs` (arma las partes OPC y las comprime). Incluye lectura
  *transitional* y *strict*, import de XLSB/ODS/XLS (valores guardados, no fórmulas), caches de
  PivotTable, DrawingML, y descifrado de workbooks protegidos (`crypto.rs`, AES/SHA-1 puro y
  WASM-safe).
- **`engine/io.rs`** husmea el contenido (XLSX/XLSB/ODS) antes de confiar en la extensión.
- Deuda conocida: el lector carga el workbook completo en memoria (sin streaming, issue #175) y
  las partes XLSX no modeladas no sobreviven a un guardado.

## 8. UI y control por agentes

- **`ui-egui`**: `SheetApp` (`crates/ui-egui/src/lib.rs:199`) es la raíz; los paneles leen estado
  (`session.doc()?.wb…`) y actúan vía `self.run(id, params)` (`lib.rs:315`), que delega en
  `Session::execute()` y resuelve los `UiRequest` (diálogos, file dialogs, activación de ribbon,
  screenshots). Incluye ribbon con keytips (Alt/F10), barra de fórmulas, grilla virtualizada
  (solo dibuja lo visible), pestañas, diálogos, task panes, tema y i18n.
- **Canal de control**: `gridcraft --control 7979`, JSON lines en localhost
  (`engine.execute`, `ui.screenshot`, `ui.click`, `ui.key`, `ui.ribbon`, `ui.resize`, …) —
  `crates/ui-egui/src/control.rs`, documentado en `docs/control-protocol.md`. Es la puerta para
  inspeccionar y operar la app real (con screenshots para verificar UI).
- **`mcp`**: servidor MCP por stdio (JSON-RPC 2.0 escrito a mano). Trait `Backend` con dos
  implementaciones: `Remote` (puentea a una app viva por el canal de control) y `Headless`
  (`Session` in-process). Herramientas: `read_range`, `write_range`, `set_cell`,
  `evaluate_formula`, `format_range`, `insert_chart`, `create_table`, `sort_range`, `filter` y
  `execute_command` (llega a todos los comandos).
- **`gridcraft-cli`**: subcomandos `eval`, `cat`, `convert`, `run`, `commands`, `functions`,
  `mcp`, `send`; usa `Session` directamente (`execute_headless`). Ejemplo:
  `gridcraft-cli run --sample budget --cmd 'home.bold={"range":"B4:F4"}' --out budget.xlsx`.
- **`gridcraft-web`**: el mismo stack sobre eframe/wgpu en WASM vía trunk.

## 9. Calidad, seguridad y testing

- **Gates de commit**: `cargo xtask ci` = fmt + clippy `-D warnings` + tests + assets + layers +
  wasm + parity check.
- **Never-crash verificado en esta revisión**: no se encontraron `unwrap()`/`expect()` en código
  de producción — los ~900+ usos de `unwrap()` están confinados a archivos de tests
  (`engine/src/tests.rs`, `xlsx/src/tests/…`, etc.); solo 5 atributos `allow(clippy::unwrap_used)`
  en todo el código de `src/`. Además: `unsafe_code = "forbid"` global, y red de seguridad final
  en `Session::execute`.
- **Fuzzing estructural**:
  - `crates/engine/tests/command_sequence_fuzz.rs`: secuencias aleatorias de comandos contra
    invariantes de undo/redo, round-trip XLSX y recálculo completo; configurable con
    `GRIDCRAFT_FUZZ_SEEDS`/`STEPS`, reproducible con `GRIDCRAFT_FUZZ_SEED`.
  - `every_command_survives_empty_params` (`crates/engine/src/tests.rs:1686`): llama **todos**
    los comandos con `{}`, `[1,2]` y rangos inválidos — ningún comando puede crashear con
    entrada hostil.
- **Tests de UI reales**: 30+ suites con `egui_kittest` (`crates/ui-egui/tests/`: keytips,
  clipboard, fórmulas por locale, celdas combinadas, contraseñas, escala de interfaz, …).
- **Assets con licencia trazada**: `cargo xtask assets` exige fila en `ATTRIBUTION.md` por cada
  asset; los íconos y muestras se generan en código (política anti-copia de vendors).
- **Logs robustos**: rotación y tope de 16 MiB, el log de un crash sobrevive al siguiente arranque
  (`README.md`, sección Logs).

## 10. Gaps y deudas conocidas

De `docs/architecture.md` ("Known architectural gaps") y `docs/gaps.md`:

1. Cálculo single-threaded (sin paralelismo en recalc).
2. Sin catálogo de texto ni text shaping completo (localización parcial; ver
   `docs/localization-parity.md`).
3. Lector XLSX sin streaming (workbook completo en memoria, #175).
4. Partes XLSX no modeladas se pierden al guardar.
5. Próximos frentes (roadmap): fidelidad XLSX en archivos reales, workbooks muy grandes,
   profundidad de gráficos, texto en todos los scripts, import `.xls` completo.

## 11. Hallazgos de esta revisión

1. **Drift en `docs/architecture.md`**: la tabla de líneas ( fechada 2026-10-10) reporta
   `engine` ~17,7k, `ui-egui` ~11,6k, `xlsx` ~10,7k; la medición actual (incluyendo tests) da
   ~29,9k / ~30,7k / ~19,4k. Puede ser crecimiento real en poco tiempo o un método de conteo
   distinto (con/sin tests). Recomendación: regenerar con `cargo xtask stats` y fijar ese comando
   como fuente canónica para el doc.
2. **Higiene never-crash sobresaliente**: la política de cero panics se cumple de verdad en
   código de producción, y está blindada por tests (fuzz + "todos los comandos con params
   hostiles"), no solo por lints.
3. **Doble registro de comandos**: catálogo de paridad (`catalog.rs`, Excel-ribbon) y comandos
   primitivos (`cell.set`, etc.) conviven; el checklist mide 274/290 sobre el catálogo. Al leer
   el código conviene no confundir "comando registrado" con "entrada de catálogo".
4. **Fortaleza estructural para agentes**: el diseño "todo es un comando" + canal de control +
   MCP + CLI significa que casi cualquier comportamiento es verificable programáticamente con
   screenshots y `engine.execute`, sin tocar la UI a mano.

## 12. Backlog sugerido para esta carpeta

Ideas de próximos documentos de análisis (uno por documento):

1. **Sistema de comandos en profundidad**: ciclo completo `cmd!` → `Session::execute` → journal/
   undo → `UiRequest` (con un comando real de punta a punta).
2. **Grafo de recálculo**: estructuras `Intervals<T>`, orden topológico, spills y ciclos.
3. **Round-trip XLSX**: qué se lee, qué se escribe, qué se pierde; el `ReadReport`.
4. **ui-egui**: ciclo de render de la grilla virtualizada, editor de fórmulas, keytips.
5. **Cobertura de tests**: mapa de qué riesgo cubre cada suite (unit vs integration vs kittest vs
   fuzz).
6. **Rendimiento**: perfil de arranque, recalc en workbook grande, límites del COW.

## Anexo · Fuentes principales

- `README.md`, `AGENTS.md`, `Cargo.toml`, `ROADMAP.md`
- `docs/architecture.md`, `docs/parity-checklist.md`, `docs/control-protocol.md` (referenciado),
  `docs/formula-input.md` (referenciado), `docs/gaps.md`, `docs/hardware-parity.md` (referenciado)
- Evidencia de código: `crates/engine/src/cmd/mod.rs`, `crates/engine/src/catalog.rs`,
  `crates/calc/src/recalc.rs`, `crates/calc/src/eval.rs`, `crates/formula/src/*`,
  `crates/xlsx/src/read.rs`, `crates/xlsx/src/write.rs`, `crates/ui-egui/src/lib.rs`,
  `crates/ui-egui/src/control.rs`, `crates/engine/src/tests.rs`,
  `crates/engine/tests/command_sequence_fuzz.rs`, `xtask/src/main.rs`
