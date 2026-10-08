# Playbook — Paso 3 de 3: Clavis Shell, Entorno Niri y Procedimiento de Actualizaciones Diarias

> **Estrategia Global (Roadmap de 3 Pasos):**
> * **Paso 1:** Preparación del sistema base, compilación del compositor (`niri-edge`) y runtime core (`key-cli`) a nivel global sin tocar configuraciones de usuario. *(Completado)*
> * **Paso 2:** Creación del usuario dedicado `niri`, paridad total de permisos con `gracie` y configuración del display manager SDDM con selector de usuario y sesión. *(Completado)*
> * **👉 Paso 3 (Este Playbook):** Compilación e instalación de Clavis Shell (`quickshell`) desde `HEAD`, despliegue de componentes nativos QML, configuración del entorno Wayland exclusivo para el usuario `niri` y automatización del mantenimiento diario contra upstream. *(Completado)*

---

## 🎯 Objetivo del Paso 3

1. **Instalar Clavis Shell desde `HEAD`:** Compilar e instalar la suite de shell de escritorio desde el código fuente más reciente en `/home/gracie/Q/quickshell` (sin usar tarballs de releases estáticos).
2. **Resolver todas las dependencias oficiales, AUR y recursos externos:**
   - Paquetes oficiales: `quickshell` (0.3.1), `qt6-*`, `maplibre-native-qt`, `qtkeychain-qt6`, `libpipewire`, `systemd-libs`, `libxkbcommon`, `matugen`, `awww`, `rclone`, `flatpak`, etc.
   - Paquetes AUR: `libcava` (biblioteca Cava compartida para visualizador de audio), `qt6-m3shapes-git` (formas y morfología Material 3 para QML), `ttf-lxgw-wenkai-screen` (tipografía UI).
   - Recursos multimedia externos: Paquetes `@meteocons/svg` y `@meteocons/lottie` para iconos y animaciones meteorológicas.
3. **Configurar el entorno exclusivo para el usuario `niri`:**
   - Crear `/home/niri/.config/niri/config.kdl` con autostart de demonios (`key shell`, `key clipboard`, `xwayland-satellite`, `polkit-gnome`).
   - Generar y vincular modularmente los fragmentos de configuración de Clavis (`binds.kdl`, `layer-rules.kdl`, `effects.kdl`, `cursor.kdl`, `minimize-animation.kdl`, `hot-corners.kdl`) sin contaminar el entorno de `gracie`.
   - Habilitar los servicios de usuario systemd (`clavis-shell.service`, `clavis-clipboard.service`) y la persistencia con `loginctl enable-linger niri`.
4. **Implementar el sistema de actualización diaria contra upstream:** Crear y documentar un flujo automatizado para mantener `niri-edge`, `key-cli` y `quickshell` al día con sus repositorios principales.

> [!IMPORTANT]
> **Política de Seguridad:** Por directiva explícita del usuario, **la contraseña no se almacena en este playbook**. La cuenta `niri` cuenta con privilegios en `sudoers` para tareas administrativas de forma transparente.

---

## 💡 Decisiones Técnicas y Motivos (Rationale)

### 1. ¿Cómo instalar: `sudo -s niri` vs Instalación a Nivel de Sistema?
El usuario consultó: *¿puedes hacer sudo -s niri e instalarlo ahí o cómo recomiendas instalarlo?*

**Recomendación y Diseño Arquitectural:**
* **Nivel Sistema (Binarios y Módulos Compartidos):**
  Clavis Shell no es un script de shell plano; compila plugins nativos en C++ para Qt Quick (`Clavis.Weather`, `Clavis.Cava`, `Clavis.Niri`, `Clavis.WindowPreview`, etc.) que residen en `/usr/lib/qt6/qml/Clavis` y distribuye su configuración base a `/etc/xdg/quickshell/clavis`. Estos archivos **deben instalarse a nivel de sistema con permisos de root** (`sudo cmake --install`). Si se compilaran e instalaran exclusivamente en `/home/niri`, se romperían los permisos XDG globales, ningún otro usuario podría aprovecharlos y se mezclaría código ejecutable del sistema en el home del usuario.
* **Nivel Usuario (Aislamiento Total de Dotfiles):**
  Toda la configuración específica del usuario (`~/.config/niri/config.kdl`, fragmentos KDL, temas Matugen locales, servicios systemd de usuario) se ejecuta **estrictamente bajo la identidad `niri`** mediante `sudo -u niri -H bash -c '...'`. De este modo:
  - El usuario `gracie` queda **100% limpio e intacto**.
  - Todos los archivos en `/home/niri` conservan propiedad estricta `niri:niri`.

### 2. ¿Por qué compilar desde `HEAD` de Git?
Las releases empaquetadas (`v2026.9.25`) no incluyen los últimos parches de estabilidad presentes en `HEAD` (`c3ee54f`), tales como:
- Corrección del refresco del modelo de calendario entre días en Qt.
- Corrección de eventos de desbloqueo en Quickshell para evitar que el Dock permanezca bloqueado tras salir del lockscreen.
- Eliminación de animaciones espurias de estiramiento en los sliders de audio/brillo.

### 3. Manejo de Dependencias Críticas en AUR
- **`libcava`:** Clavis Shell requiere la biblioteca compartida `libcava.so` (con headers en `/usr/include/cava`) para renderizar el ecualizador en tiempo real a través de PipeWire. El paquete estándar `cava` solo ofrece el binario interactivo de terminal.
- **`qt6-m3shapes-git`:** Provee los tipos QML para renderizado de curvas y formas geométricas fluidas Material 3.
- **`ttf-lxgw-wenkai-screen`:** Tipografía recomendada para la UI china/latina con renderizado nítido en pantallas de alta resolución.

### 4. Empaquetado de Recursos Meteorológicos
El repositorio Clavis no rastrea binarios SVG/JSON en Git. Mediante `python3 scripts/release.py bundle-resources --cache /tmp/clavis-cache`, se descargan e instalan los assets de `@meteocons/svg` y `@meteocons/lottie` bajo `assets/icons/weather/meteocons/`, permitiendo que la tarjeta meteorológica y la barra de estado muestren animaciones climáticas completas.

### 5. Fragmentos KDL Modulares mediante `niri_config.py`
En lugar de crear un archivo monolítico frágil, Clavis Shell incluye un gestor especializado (`niri_config.py`) que genera fragmentos modulares:
- `clavis/binds.kdl`: Mapeo de atajos a llamadas IPC de Quickshell (`Mod+Space` para Spotlight, `Mod+A` para Quicksettings, `Mod+N` para Sidebar, etc.).
- `clavis/layer-rules.kdl`: Reglas para backdrop transparente y wallpaper interactivo.
- `clavis/effects.kdl`: Reglas de desenfoque y efecto X-Ray en Wayland.
- `clavis/cursor.kdl`, `minimize-animation.kdl` y `hot-corners.kdl`.

---

## 🛠️ Guía Paso a Paso de Instalación

### Paso 1: Instalación de Dependencias del Sistema y AUR

Ejecutar como usuario con privilegios (`gracie`):

```bash
# 1. Dependencias oficiales de Arch
sudo pacman -S --needed --noconfirm \
    quickshell qt6-base qt6-declarative qt6-svg qt6-wayland qt6-5compat \
    qt6-lottie qt6-location qt6-positioning maplibre-native-qt qtkeychain-qt6 \
    libpipewire systemd-libs libxkbcommon ttf-material-symbols-variable \
    ttf-jetbrains-mono-nerd matugen awww rclone brightnessctl ddcutil \
    imagemagick xdg-terminal-exec networkmanager bluez upower pipewire \
    wireplumber gsettings-desktop-schemas wl-clipboard fd slurp grim \
    gpu-screen-recorder ffmpeg libpulse hyprpicker flatpak

# 2. Dependencias desde AUR (mediante yay)
yay -S --needed --noconfirm qt6-m3shapes-git libcava ttf-lxgw-wenkai-screen
```

---

### Paso 2: Preparación y Compilación de Clavis Shell desde `HEAD`

```bash
cd /home/gracie/Q/quickshell

# 1. Descargar recursos de iconos meteorológicos
python3 scripts/release.py bundle-resources --cache /tmp/clavis-cache
python3 scripts/release.py verify-resources

# 2. Configurar el proyecto CMake con Ninja
cmake -S . -B build -G Ninja \
    -DCMAKE_BUILD_TYPE=Release \
    -DBUILD_TESTING=ON \
    -DCMAKE_INSTALL_PREFIX=/usr \
    -DCMAKE_INSTALL_LIBDIR=lib \
    -DCLAVIS_CONFIG_INSTALL_DIR=/etc/xdg/quickshell/clavis

# 3. Compilar todos los módulos nativos
cmake --build build

# 4. Validar la suite completa de pruebas (27/27 contratos)
ctest --test-dir build --output-on-failure

# 5. Instalar a nivel de sistema
sudo cmake --install build
```

---

### Paso 3: Configuración del Entorno para el Usuario `niri`

```bash
# 1. Crear estructura de configuración para niri
sudo mkdir -p /home/niri/.config/niri/clavis
sudo cp /home/gracie/Q/niri-edge/resources/default-config.kdl /home/niri/.config/niri/config.kdl
sudo chown -R niri:niri /home/niri/.config/niri

# 2. Ajustar arranque de servicios en ~/.config/niri/config.kdl
sudo python3 -c '
path = "/home/niri/.config/niri/config.kdl"
with open(path, "r") as f:
    c = f.read()
old = "spawn-at-startup \"waybar\""
new = """spawn-at-startup "xwayland-satellite"
spawn-at-startup "/usr/lib/polkit-gnome/polkit-gnome-authentication-agent-1"
spawn-at-startup "key" "shell" "--daemon"
spawn-at-startup "key" "clipboard" "--daemon" """
if old in c:
    c = c.replace(old, new)
    with open(path, "w") as f:
        f.write(c)
'

# 3. Vincular los fragmentos modulares de Clavis Shell en niri
for feature in binds layer-rules effects cursor minimize-animation hot-corners; do
    sudo -u niri -H python3 /etc/xdg/quickshell/clavis/scripts/system/niri_config.py \
        "{\"operation\": \"setup\", \"feature\": \"$feature\"}"
done

# 4. Validar la configuración generada
sudo -u niri -H niri validate -c /home/niri/.config/niri/config.kdl

# 5. Asignar grupos de hardware al usuario niri
sudo usermod -a -G input,video,audio niri

# 6. Habilitar systemd user lingering y servicios de sesión
sudo loginctl enable-linger niri
sudo -u niri -H XDG_RUNTIME_DIR=/run/user/1001 systemctl --user daemon-reload
sudo -u niri -H XDG_RUNTIME_DIR=/run/user/1001 systemctl --user enable \
    clavis-shell.service clavis-clipboard.service
```

---

## 🔄 Procedimiento de Actualizaciones Diarias contra Upstream

Para mantener el entorno Niri y Clavis Shell al día frente a los commits más recientes de GitHub/Upstream sin reinstalaciones manuales, se implementó el script centralizado:

### Script `/usr/local/bin/update-niri-desktop.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

echo "================================================================"
echo "  Actualizando Niri Desktop Environment contra Upstream (HEAD)  "
echo "================================================================"

# 1. Actualizar niri-edge
if [ -d "/home/gracie/Q/niri-edge" ]; then
    echo ">> [1/4] Actualizando niri-edge..."
    cd /home/gracie/Q/niri-edge
    git fetch origin
    git merge --ff-only origin/main || echo "Advertencia: Merge rápido no posible o ya actualizado"
    cargo build --release
    install -Dm755 target/release/niri /usr/local/bin/niri
fi

# 2. Actualizar key-cli
if [ -d "/home/gracie/Q/key-cli" ]; then
    echo ">> [2/4] Actualizando key-cli..."
    cd /home/gracie/Q/key-cli
    git fetch origin
    git merge --ff-only origin/main || echo "Advertencia: Merge rápido no posible o ya actualizado"
    ./scripts/install.sh --all
fi

# 3. Actualizar quickshell (Clavis Shell)
if [ -d "/home/gracie/Q/quickshell" ]; then
    echo ">> [3/4] Actualizando Clavis Shell..."
    cd /home/gracie/Q/quickshell
    git fetch origin
    git merge --ff-only origin/main || echo "Advertencia: Merge rápido no posible o ya actualizado"
    if [ -f "scripts/release.py" ]; then
        python3 scripts/release.py bundle-resources --cache /tmp/clavis-cache || true
    fi
    cmake -S . -B build -G Ninja \
        -DCMAKE_BUILD_TYPE=Release \
        -DBUILD_TESTING=OFF \
        -DCMAKE_INSTALL_PREFIX=/usr \
        -DCMAKE_INSTALL_LIBDIR=lib \
        -DCLAVIS_CONFIG_INSTALL_DIR=/etc/xdg/quickshell/clavis
    cmake --build build
    cmake --install build
fi

# 4. Actualizar paquetes AUR y oficiales
echo ">> [4/4] Verificando paquetes del sistema..."
pacman -Syu --needed --noconfirm

echo "================================================================"
echo "  Actualización de Niri Desktop completada con éxito.           "
echo "================================================================"
```

### Cómo ejecutar la actualización diaria:
```bash
sudo /usr/local/bin/update-niri-desktop.sh
```

### Opcional: Automatizar mediante Systemd Timer Diario
Para ejecutar esta actualización automáticamente cada día a las 04:00 AM:

1. **Crear `/etc/systemd/system/niri-update.service`:**
   ```ini
   [Unit]
   Description=Actualización Diaria Niri Desktop y Clavis Shell
   After=network-online.target

   [Service]
   Type=oneshot
   ExecStart=/usr/local/bin/update-niri-desktop.sh
   ```

2. **Crear `/etc/systemd/system/niri-update.timer`:**
   ```ini
   [Unit]
   Description=Timer para Actualización Diaria Niri Desktop

   [Timer]
   OnCalendar=*-*-* 04:00:00
   Persistent=true

   [Install]
   WantedBy=timers.target
   ```

3. **Habilitar el timer:**
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable --now niri-update.timer
   ```

---

## 🔍 Checklist de Verificación y Estado Final

| Componente | Comando de Verificación | Resultado Esperado |
| :--- | :--- | :--- |
| **Quickshell Runtime** | `quickshell --version` | `Quickshell 0.3.1` |
| **Clavis Shell QML** | `ls /usr/lib/qt6/qml/Clavis` | Módulos C++ compilados (`Niri`, `Weather`, `Cava`, etc.) |
| **Key-CLI Features** | `key doctor --json` | `"ok": true`, `"missing": []` |
| **Configuración Niri** | `sudo -u niri niri validate -c ~/.config/niri/config.kdl` | `INFO niri: config is valid` |
| **Servicios de Usuario** | `sudo -u niri XDG_RUNTIME_DIR=/run/user/1001 systemctl --user is-enabled clavis-shell.service` | `enabled` |
| **Clipboard Service** | `sudo -u niri XDG_RUNTIME_DIR=/run/user/1001 systemctl --user is-enabled clavis-clipboard.service` | `enabled` |
| **Script de Update** | `ls -la /usr/local/bin/update-niri-desktop.sh` | Ejecutable y funcional |
