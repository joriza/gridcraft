# 04 · Consultas de uso — registro creciente

| | |
| --- | --- |
| **Commit base** | `3455bed` (docs/analisis 03 ampliado con `eval --in`) |
| **Fecha de creación** | 2026-10-12 |
| **Propósito** | Registro de las consultas de uso realizadas sobre GridCraft y sus respuestas. Se agrega cada consulta nueva con número secuencial |

**Convención**: cada consulta queda numerada con fecha, la pregunta textual, una respuesta
breve y la evidencia (comando ejecutado con su resultado real, o referencia al documento
temático donde está el detalle completo).

---

## C1 · ¿Es posible ejecutar el proyecto? — 2026-10-11

**Pregunta**: ¿Se puede correr GridCraft en esta máquina?

**Respuesta**: Sí. Requirió instalar una sola vez el toolchain (Rust 1.99.0 estable + VS Build
Tools 2022 con workload C++). Verificados los tres objetivos: desktop, CLI y web.

**Detalle**: [02-como-iniciar-la-aplicacion.md](02-como-iniciar-la-aplicacion.md) §1-2.

## C2 · ¿Cómo pruebo la aplicación localmente? — 2026-10-12

**Pregunta**: No entendí cómo hacer para probar la aplicación localmente.

**Respuesta**: Tres pasos en PowerShell (terminal **nueva**, para que tome el PATH de Rust):
`cd D:\Desarrollo\gridcraft` → `cargo run --release -p gridcraft -- --sample sales`. La primera
vez compila 5-15 min; después abre en segundos. Sin `--sample` abre vacío; con un archivo como
argumento lo abre directo.

**Detalle**: [02-como-iniciar-la-aplicacion.md](02-como-iniciar-la-aplicacion.md) §2-3.

## C3 · ¿Funciona la versión web? — 2026-10-12

**Pregunta**: Probá la versión web. / Fue una prueba rápida, parece comportarse igual.

**Respuesta**: Sí — compilada con trunk 0.21.14 (build 2m22s), servida en
`http://127.0.0.1:8771/?sample=sales` y confirmada por el usuario en el navegador. El motor y
la UI son el mismo código Rust; lo que cambia es la plataforma (render WebGL2/WebGPU, archivos
por file picker, sin canal de control ni MCP en web).

**Detalle**: [02-como-iniciar-la-aplicacion.md](02-como-iniciar-la-aplicacion.md) §6.

## C4 · ¿Cómo funciona el CLI y cuánto ocupa? — 2026-10-12

**Pregunta**: Contame brevemente cómo funciona el CLI. ¿Cuánto ocuparía en disco y en memoria
tener solo el CLI?

**Respuesta**: Binario sin UI que habla directo con el motor y ejecuta los mismos comandos que
la app. Medido: **12,9 MB** en disco (release), **~17 MB de RAM** en reposo (servidor MCP
esperando), ~40× menos que la app desktop. Un solo `.exe` autocontenido.

**Detalle**: [03-uso-del-cli.md](03-uso-del-cli.md) §1 y §3.

## C5 · ¿Cómo se usa el CLI en la práctica? — 2026-10-12

**Pregunta**: ¿Cómo sería el uso del CLI?

**Respuesta**: Siete flujos verificados: `eval` (calculadora de fórmulas), `info`/`cat`
(inspección), `convert` (xlsx↔csv/tsv/json/html), `run` (ediciones batch con `--cmd`/`--script`),
`commands`/`functions` (catálogo), `check` (verificación de integridad), `mcp`/`send` (modo
agente).

**Detalle**: [03-uso-del-cli.md](03-uso-del-cli.md) §2.

## C6 · ¿Llevándome el exe a otro equipo alcanza? — 2026-10-12

**Pregunta**: Para utilizarlo en otro equipo, ¿con solo llevarme el exe alcanza?

**Respuesta**: Casi siempre sí. Según `dumpbin`, el exe dinámico solo depende de DLLs de
Windows (más `VCRUNTIME140`, presente en casi cualquier equipo actualizado). Se generó además
un **exe autónomo** con CRT estático (`dist\cli\gridcraft-cli-standalone.exe`, 13,1 MB) que no
necesita instalar nada jamás. Limitaciones: solo Windows x64 (ARM64 por emulación); para
macOS/Linux hay builds oficiales por plataforma.

**Detalle**: [03-uso-del-cli.md](03-uso-del-cli.md) §4-5.

## C7 · ¿Puedo usar coordenadas de un documento Excel con `eval`? — 2026-10-12

**Pregunta**: En `eval '=SUM(1,2,3)'` estoy cargando los datos a mano, ¿puedo utilizar las
coordenadas de un documento de Excel? Es como una forma de usar las fórmulas sin abrir el
archivo en Excel.

**Respuesta**: Exactamente eso. `--in <archivo>` resuelve las coordenadas contra los datos
reales; `--sheet`, `--cell` y `--json` afinan. Verificado sobre un archivo generado por CLI
(A1=10, A2=20, A3=32, B1=`=SUM(A1:A3)*2`): `eval '=SUM(A1:A3)' --in` → `62`;
`eval '=B1' --in` → `124`. Excel no interviene en ningún momento: el motor propio de GridCraft
lee y evalúa. Para que la fórmula quede **guardada** en el archivo:
`run --in libro.xlsx --cmd 'cell.set={"cell":"D1","input":"=SUM(A1:A3)"}' --out libro.xlsx`.

**Evidencia**: transcripción completa en [03-uso-del-cli.md](03-uso-del-cli.md) §2 (eval).

---

## Consultas nuevas

Cuando surja una consulta nueva, se agrega acá con el número secuencial siguiente (C8, C9, …),
fecha, pregunta, respuesta breve y evidencia.
