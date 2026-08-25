# 📌 Contexto del Proyecto: Spoolman Pressure Advance Auto-Calibration (Snapmaker U1 Extended Firmware)

> **Documento Handoff para Sesiones Antigravity / Agentic Coding**  
> *Creado para mantener la continuidad del desarrollo al migrar entre entornos (Windows/WSL -> Linux Native).*

---

## 🚀 Resumen del Proyecto

Este proyecto añade soporte bidireccional y auto-calibración de **Pressure Advance (PA)** sensible al **tamaño y tipo de boquilla (Nozzle)** integrado con **Spoolman** en el firmware extendido de la impresora 3D **Snapmaker U1** (`SnapmakerU1-Extended-Firmware`).

- **Repositorio Upstream:** `paxx12-snapmaker-u1/SnapmakerU1-Extended-Firmware` (rama base por defecto: `develop`)
- **Fork del Proyecto:** `pechex/SnapmakerU1-Extended-Firmware`
- **Ubicación de la Mod:** `overlays/mods/pechex/01-spoolman-pa-auto/`
- **Perfil de Compilación:** `extended-pechex` (`DOCKER=1 ./dev.sh make build PROFILE=extended-pechex`)

---

## 💡 Funcionalidades Desarrolladas

### 1. Sincronización y Aplicación de PA Sensible a Nozzle (`pa_matrix`)
- **Estructura indexada en Spoolman:** El PA se guarda en Spoolman dentro de un campo dinámico JSON personalizado llamado `pa_matrix` (ejemplo: `{"0.4_standard": 0.0450, "0.2_standard": 0.0620}`), indexado por el diámetro del nozzle (`nozzle_diameter`) y su variante (`standard` / `high_flow`).
- **Detección dinámica de Nozzle:** Moonraker consulta el tamaño del nozzle en tiempo real para cada cabezal a través del componente nativo de Moonraker (`machine.get_product_info()`), el cual lee `/oem/printer_data/.lava.sn` (configurado en la pantalla táctil de la Snapmaker U1).
- **Guardado al calibrar:** Al concluir una calibración de flujo (`FLOW_CALIBRATE`), Klipper envía el canal (`channel=extruder_index`) mediante la RPC `spoollink_save_pa`. Moonraker detecta el nozzle activo en ese cabezal y actualiza la clave correspondiente en `pa_matrix`.
- **Aplicación al cargar:** Al resolver/cargar un filamento registrado en Spoolman, Moonraker busca en `pa_matrix` la entrada correspondiente al diámetro y tipo de boquilla montada en ese cabezal. Si existe, la envía como `SPOOLMAN_PA` y Klipper la aplica directamente al extrusor (`estepper._set_pressure_advance`). Si no existe, envía `None`.

### 2. Auto-Calibración Opcional al Cargar Filamento (Feature Avanzada)
- **Toggle en Web UI:** Opción `spoolman_auto_pa` visible en la pantalla `/firmware-config/` (activable/desactivable en la interfaz web de la impresora).
- **Disparo automático estricto por comando LOAD:** Se intercepta el comando `AUTO_FEEDING ... LOAD=1` para registrar exactamente qué canal/extrusor fue seleccionado para la maniobra de carga en `pending_load_channels`. Únicamente se calibran aquellos extrusores que fueron explícitamente cargados y que **no poseen** un valor de PA guardado para esa boquilla (`SPOOLMAN_PA` es `None` o `<= 0`). Los filamentos previamente cargados en otros extrusores son ignorados.
- **Estabilización de multicarga (2 segundos):** Un temporizador asíncrono en el reactor de Klipper aguarda a que concluyan todas las maniobras de carga física antes de iniciar la calibración.
- **Conmutación automática de cabezal:** Cambia automáticamente de cabezal activo (`T0`-`T3`) antes de ejecutar `FLOW_CALIBRATE`.
- **Escudo de seguridad anti-bucle:** Limita los intentos de auto-calibración a máximo **1 intento por sesión de carga de bobina** (`self.calibration_attempts`), impidiendo bucles infinitos ante errores de red o caídas de servidor HTTP de Spoolman.
- **Notificación y estado de pantalla:** Cambia el estado de pantalla a `FLOW_CALIBRATION` e informa en la consola y pantalla `M117 Calibrating extrusion T<canal>...`.

---

## 🛠️ Estructura de Archivos y Parches

Todos los parches están aislados dentro de `overlays/mods/pechex/01-spoolman-pa-auto/`:

```text
overlays/mods/pechex/01-spoolman-pa-auto/
├── README.md
├── patches/
│   └── home/
│       └── lava/
│           ├── klipper/
│           │   ├── 01-flow_calibrator.patch   # Envía channel=extruder_index en RPC spoollink_save_pa
│           │   └── 02-spoollink.patch         # Lógica SpoolLink Klipper (auto-PA, hooks, anti-loop, T-switch)
│           └── moonraker/
│               └── 01-spoollink.patch         # Moonraker SpoolLink (pa_matrix JSON en Spoolman, machine nozzle lookup)
└── root/
    └── usr/
        └── local/
            └── share/
                └── firmware-config/
                    └── functions/
                        └── 28_settings_spoolman_auto_pa.yaml  # Configuración Web UI /firmware-config/
```

---

## 🌿 Estado de las Ramas en Git

1. **`feature/spoolman-pa-clean-pr`**:
   - Rama reubicada directamente sobre `upstream/develop` con la estructura modular para el **Pull Request**.
   - **Commit 1:** `feat(spoollink): Add Spoolman Pressure Advance integration (save on calibration, apply on load)`
   - **Commit 2:** `feat(spoollink): Add optional automatic Pressure Advance calibration on filament load`
   - **Commit 3:** `ci: Update pull_request workflow to build with PROFILE=extended-pechex`

2. **`feature/spoolman-pa-testing`** / **`pa-testing`**:
   - Rama activa de pruebas que incluye la estructura `pa_matrix` sensible a boquillas (`nozzle_diameter`), avisos de pantalla (`_set_screen_state`, `M117`) y compatibilidad con `machine` de Moonraker.

---

## 🧪 Comandos Útiles para Sesiones en Linux

### Compilación del Firmware (Docker)
```bash
cd /path/to/SnapmakerU1-Extended-Firmware
DOCKER=1 ./dev.sh make build PROFILE=extended-pechex OVERWRITE=1
```

### Verificación de Sintaxis de Parches (Dry-Run)
```bash
patch --dry-run -F 0 -p1 -d overlays/firmware-extended/38-feature-spoollink/root/home/lava/klipper < overlays/mods/pechex/01-spoolman-pa-auto/patches/home/lava/klipper/02-spoollink.patch
patch --dry-run -F 0 -p1 -d overlays/firmware-extended/38-feature-spoollink/root/home/lava/moonraker < overlays/mods/pechex/01-spoolman-pa-auto/patches/home/lava/moonraker/01-spoollink.patch
```

### Subir Cambios a GitHub (si fuera necesario)
```bash
git push -u origin feature/spoolman-pa-testing --force
```

---
*Este documento contiene todo el mapa de arquitectura, decisiones de diseño y estado de ramas para continuar el desarrollo sin perder contexto.*
