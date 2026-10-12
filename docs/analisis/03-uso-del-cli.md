# 03 · Uso del CLI y su huella — GridCraft

| | |
| --- | --- |
| **Commit base** | `5218368` (docs/analisis 02 con sección web verificada) |
| **Fecha** | 2026-10-12 |
| **Verificado en** | Esta máquina: Windows 11 x64, Rust 1.99.0 (MSVC). Todos los comandos y mediciones se ejecutaron realmente |
| **Binarios medidos** | `target\release\gridcraft-cli.exe` (dinámico) y `dist\cli\gridcraft-cli-standalone.exe` (estático) |

---

## 1. Qué es y cómo funciona

`gridcraft-cli` es un binario delgado (`apps/gridcraft-cli`, ~2.200 líneas) que habla **directo
con el motor**, sin capa gráfica:

1. **Sin UI**: no compila egui/winit/wgpu — solo las capas del motor (L0-L5: core, numfmt,
   formula, functions, model, calc, xlsx, engine). Por eso compila en ~1 minuto vs minutos de la
   app desktop.
2. **Los mismos comandos que la app**: cada subcomando crea una `Session` en memoria y ejecuta
   por `execute_headless()` los mismos comandos del catálogo (`home.bold`, `cell.set`, …). Cero
   lógica duplicada: UI, CLI, MCP y canal de control comparten semántica exacta.
3. **Puro stdin/stdout**: texto adentro, texto afuera. Sin ventana, sin GPU, sin diálogos.
4. **Modo servidor**: `mcp` expone el motor como servidor MCP (JSON-RPC 2.0 por stdio) y
   `send <puerto>` opera una app desktop viva por su canal de control.

## 2. Los flujos de uso (todos verificados)

### Calculadora con funciones de Excel

```powershell
gridcraft-cli eval '=SUM(1,2,3)'                      # → 6
gridcraft-cli eval '=XLOOKUP("b",{"a","b"},{1,2})'    # → 2
gridcraft-cli eval '=PMT(0.05/12, 60, -20000)'        # cuota de un préstamo
gridcraft-cli eval '=SUM(A1:A3)' --in libro.xlsx      # sobre coordenadas de un archivo real
gridcraft-cli eval '=SUM(A1:A3)' --in libro.xlsx --sheet Ventas   # hoja específica
gridcraft-cli eval '=A1*1.21'    --in libro.xlsx --cell B2       # celda de contexto
gridcraft-cli eval '=B1'         --in libro.xlsx --json           # salida estructurada
```

Verificado sobre un archivo generado por CLI (A1=10, A2=20, A3=32, B1=`=SUM(A1:A3)*2`):
`eval '=SUM(A1:A3)' --in` → `62`; `eval '=B1' --in` → `124` (evalúa la fórmula guardada).
El archivo nunca se abre en Excel: el motor propio de GridCraft lo lee y evalúa.

### Inspeccionar archivos

```powershell
gridcraft-cli info libro.xlsx                 # hojas, rangos usados, tablas, gráficos, nombres
gridcraft-cli cat libro.xlsx --range B4:F15   # rango como tabla de texto alineada
gridcraft-cli cat libro.xlsx --formulas       # fórmulas en vez de valores
```

### Convertir formatos (por extensión del destino)

```powershell
gridcraft-cli convert reporte.xlsx reporte.csv    # también .tsv, .json, .html
gridcraft-cli convert datos.csv datos.xlsx        # y viceversa
```

### Automatizar ediciones (batch, sin ventana)

```powershell
gridcraft-cli run --in libro.xlsx --cmd 'home.bold={"range":"A1:C1"}' `
  --cmd 'cell.set={"cell":"D1","input":"=SUM(A1:C1)"}' --out libro.xlsx
gridcraft-cli run --sample sales --print B4:F8        # salida verificada: tabla formateada
```

`--script archivo.jsonl` acepta la misma secuencia en un archivo (un JSON por línea), igual que
el canal de control de la app desktop.

### Consultar el catálogo

```powershell
gridcraft-cli commands --search sort       # comandos + documentación de params (verificado)
gridcraft-cli functions --search XLOOKUP   # funciones por nombre
gridcraft-cli functions --category date    # funciones por categoría
```

### Verificar integridad

```powershell
gridcraft-cli check libro.xlsx
```

Recalcula todo el libro y compara con los valores cacheados + verifica la fidelidad del
round-trip XLSX (sale con código 1 si hay diferencias). Útil para detectar archivos corruptos o
generados por terceros con problemas.

### Modo agente

```powershell
gridcraft-cli mcp                      # servidor MCP por stdio (Claude, otros clientes)
gridcraft-cli mcp --connect 7979       # puenteado a una app desktop viva
gridcraft-cli send 7979 ui.screenshot '{"path":"captura.png"}'   # comando suelto a la app
```

## 3. Huella medida

| Recurso | Valor | Nota |
| --- | --- | --- |
| Disco (exe dinámico, release) | **12,9 MB** | `target\release\gridcraft-cli.exe` |
| Disco (exe estático, release) | **13,1 MB** | `dist\cli\gridcraft-cli-standalone.exe` |
| RAM en reposo | **~17 MB** (medido) | Servidor MCP esperando comandos |
| RAM vs app desktop | ~40× menos | La desktop andaba en ~690 MB (debug, con GPU y UI) |
| Tiempo de compilación | ~55 s (release, deps cacheadas) | Sin la UI es lo más rápido de compilar |

Con un workbook chico cargado el consumo queda en ese orden; escala con el tamaño del libro (el
modelo copy-on-write de `model` lo hace eficiente).

## 4. Portabilidad del exe — qué encontró `dumpbin`

Dependencias del exe **dinámico** (el build normal): `KERNEL32`, `ntdll`, `WS2_32`,
`bcryptprimitives`, `VCRUNTIME140` y las `api-ms-win-crt-*` (Universal CRT). Todo es DLL de
Windows excepto `VCRUNTIME140`, que está presente en prácticamente cualquier Windows 10/11
actualizado (la instalan miles de programas); en una instalación limpia puede faltar y se
resuelve con el "Visual C++ Redistributable x64" (una vez).

Dependencias del exe **estático** (crt-static): solo `KERNEL32`, `ntdll`, `WS2_32`,
`bcryptprimitives` y `api-ms-win-core-synch-l1-2-0` — **todas núcleo del sistema operativo**.
Corre en cualquier Windows 10/11 x64 sin instalar absolutamente nada.

| Escenario | Dinámico | Estático |
| --- | --- | --- |
| Windows 10/11 x64 normal | Sí | Sí |
| Windows limpio sin VC++ Redistributable | Puede fallar | **Sí** |
| Windows ARM64 | Sí (emulación x64) | Sí (emulación x64) |
| Windows 32 bits | No | No |
| macOS / Linux | No: usar el build de esa plataforma (las releases oficiales traen `gridcraft-cli` por OS) | Igual |

## 5. Cómo se generó el exe autónomo

```powershell
$env:CARGO_TARGET_DIR = "target-static"
$env:RUSTFLAGS = "-C target-feature=+crt-static"
cargo build --release -p gridcraft-cli
```

El resultado quedó copiado en `dist\cli\gridcraft-cli-standalone.exe` (13,1 MB). Verificado:
sin `VCRUNTIME140` según `dumpbin /DEPENDENTS`, y `eval '=SUM(1,2,3)'` → `6`. La carpeta
`dist\` está en `.gitignore` (igual que `target-*/`), así que el binario no entra al repo.

Para tenerlo siempre en el PATH de esta máquina: `cargo install --path apps\gridcraft-cli`.

## 6. Referencias

- `docs/cli.md` (referencia oficial del CLI), `docs/mcp.md`, `docs/control-protocol.md`
- `apps/gridcraft-cli/src/` (código), `crates/engine/src/cmd/` (catálogo de comandos)
- Guías previas: [02-como-iniciar-la-aplicacion.md](02-como-iniciar-la-aplicacion.md)
