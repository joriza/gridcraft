# 02 · Cómo iniciar la aplicación — GridCraft en Windows

| | |
| --- | --- |
| **Commit base** | `a8285fc` (docs/analisis 01) |
| **Fecha** | 2026-10-12 |
| **Verificado en** | Esta máquina: Windows, Rust 1.99.0 (MSVC), VS Build Tools 2022 17.14, GPU AMD Radeon integrada (DirectX 12) |
| **Método** | Todos los comandos de esta guía se ejecutaron realmente; los tiempos están medidos |

---

## 1. Requisitos previos (una sola vez)

Ya instalados en esta máquina (ver análisis del entorno):

1. **Rust stable ≥ 1.90** vía rustup → `rustc --version` tiene que responder 1.99.0 o más.
2. **VS Build Tools con workload C++ (MSVC)** → es el enlazador; sin esto el build falla al linkear.

Verificación rápida:

```powershell
cargo --version        # Cargo ≥ 1.99
rustc --version        # rustc 1.99.0
```

## 2. Inicio rápido — la app desktop en 3 pasos

```powershell
# 1) Abrir una terminal NUEVA (tecla Windows → "powershell" → Enter)
#    (las terminales abiertas antes de instalar Rust no conocen `cargo`)

# 2) Entrar al proyecto
cd D:\Desarrollo\gridcraft

# 3) Compilar y abrir la app con el tablero de ejemplo
cargo run --release -p gridcraft -- --sample sales
```

- **La primera vez tarda 5-15 min** (compila todo el workspace optimizado: ~120k líneas + egui/wgpu). Las siguientes veces abre en segundos.
- La ventana que se abre es la planilla completa: ribbon, barra de fórmulas, grilla, gráficos. Se usa como Excel.

### Sin el `--sample`

```powershell
cargo run --release -p gridcraft                # workbook vacío
cargo run --release -p gridcraft -- libro.xlsx  # abre un archivo propio
```

## 3. Opciones de arranque de la app desktop

Flags reales, verificados contra `apps/gridcraft/src/main.rs`:

| Flag / argumento | Efecto | Ejemplo |
| --- | --- | --- |
| `--sample NOMBRE` | Abre un workbook de ejemplo incluido: `sales`, `budget`, `grades` | `gridcraft --sample budget` |
| *(archivo posicional)* | Abre ese archivo (.xlsx, .csv, …) | `gridcraft C:\datos\libro.xlsx` |
| `--control PUERTO` | Abre el **canal de control JSON** en localhost — permite manejar la app con comandos, capturar screenshots, simular teclado/mouse | `gridcraft --sample sales --control 7979` |
| `--version` | Muestra la versión | `gridcraft --version` |

> **Ojo:** la app **no soporta `--help`** — un flag desconocido no imprime ayuda: la app arranca
> igual (verificado). La documentación de flags es esta tabla y el código de `main.rs`.

### Release vs debug

| Perfil | Comando | Cuándo usarlo |
| --- | --- | --- |
| `--release` | `cargo run --release -p gridcraft` | Uso real: fluido (LTO activado) |
| debug (default) | `cargo run -p gridcraft` | Desarrollo: compila 3-4× más rápido, UI algo menos fluida |

Tiempos medidos en esta máquina (build incremental, deps ya cacheadas): binario desktop debug
**1m 36s**; CLI debug **1m 43s** desde cero; ejemplo snapshot **3m**.

## 4. CLI headless — usar el motor sin ventana

El binario `gridcraft-cli` expone todo el motor por consola (subcomandos verificados con
`--help`; acá no hay que compilar la UI, así que es el smoke test más rápido):

```powershell
cargo run -p gridcraft-cli -- eval '=SUM(1,2,3)'                     # → 6  (verificado)
cargo run -p gridcraft-cli -- cat libro.xlsx --range B4:F15          # tabla de texto alineada
cargo run -p gridcraft-cli -- convert entrada.xlsx salida.csv        # conversión por extensión
cargo run -p gridcraft-cli -- run --sample budget --cmd 'home.bold={{"range":"B4:F4"}}' --out out.xlsx
cargo run -p gridcraft-cli -- commands --search chart                # lista comandos del engine
cargo run -p gridcraft-cli -- functions --search XLOOKUP             # busca funciones
cargo run -p gridcraft-cli -- check libro.xlsx                       # oráculo de valores + round-trip
```

Lista completa de subcomandos: `info`, `convert`, `eval`, `cat`, `check`, `run`, `commands`,
`functions`, `mcp`, `send`, `version`.

Para no recompilar en cada uso, el binario queda en `target\debug\gridcraft-cli.exe`
(o `target\release\` si se compiló con `--release`) y puede ejecutarse directo.

## 5. Render offscreen (sin abrir ventana)

Ideal para verificar UI desde scripts o agentes — genera un PNG de la app:

```powershell
cargo run -p gridcraft-ui-egui --example snapshot -- --sample sales C:\temp\captura.png
```

Verificado: produce un PNG de 2880×1800 con ribbon, tabla estilizada, data bars, sparklines y
gráficos renderizados por la GPU real.

## 6. Versión web (WASM) — aún no instalada en esta máquina

Mismo código Rust compilado a `wasm32`. Requiere dos pasos previos (no hechos todavía acá):

```powershell
rustup target add wasm32-unknown-unknown   # target WASM
scoop install trunk                        # o: cargo install trunk

cd apps\gridcraft-web
trunk serve --release                      # → http://127.0.0.1:8771
```

## 7. Solución de problemas

| Síntoma | Causa | Solución |
| --- | --- | --- |
| `cargo no se reconoce…` | Terminal abierta antes de instalar Rust | Cerrala y abrí una **nueva** (el PATH se carga al abrir) |
| El primer `cargo run` tarda mucho | Compila ~120k líneas + dependencias | Normal una sola vez; los siguientes builds son incrementales |
| `error: linker 'link.exe' not found` | Falta VS Build Tools C++ | Instalar workload `Microsoft.VisualStudio.Workload.VCTools` |
| La ventana no abre o va lenta | Problema de driver gráfico | Probar `$env:WGPU_BACKEND = "gl"` (o `"vulkan"`) antes de ejecutar; ver README |
| ¿Cómo sé qué GPU está usando? | — | Aparece en el log al arrancar: `graphics: DirectX 12 adapter "AMD Radeon(TM) Graphics"` |

**Logs**: la app escribe en `%APPDATA%\GridCraft\logs\gridcraft.log` (rota a `.1.log`, `.2.log`;
tope 16 MiB). Si la app crashea, ese log sobrevive para el reporte de bug. Nivel de log
ajustable con `RUST_LOG` (p. ej. `RUST_LOG=debug`).

## 8. Manejar la app por programación (canal de control)

Con `--control 7979`, la app acepta JSON lines en `127.0.0.1:7979` — así la manejan agentes y
tests:

```json
{"id":1,"method":"engine.execute","params":{"command":"cell.set","params":{"cell":"B2","input":"=SUM(A1:A9)"}}}
{"id":2,"method":"ui.screenshot","params":{"path":"captura.png"}}
```

También desde el CLI: `gridcraft-cli send 7979 engine.execute '{"command":"home.bold"}'`.
Protocolo completo: `docs/control-protocol.md`; servidor MCP: `gridcraft-cli mcp` y `docs/mcp.md`.

## 9. Referencias

- `README.md` (secciones Quick start, Logs, For agents, problemas de GPU en Windows)
- `docs/cli.md`, `docs/control-protocol.md`, `docs/mcp.md`
- Código de flags: `apps/gridcraft/src/main.rs`
- Análisis previo: [01-analisis-general-del-codigo.md](01-analisis-general-del-codigo.md)
