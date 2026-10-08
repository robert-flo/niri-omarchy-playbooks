# Playbook — Paso 1 de 3: Instalación de niri-edge y key-cli (Nivel Sistema)

> **Estrategia Global (Roadmap de 3 Pasos):**
> * **👉 Paso 1 (Este Playbook):** Preparación del sistema base, compilación e instalación del compositor (`niri-edge`), herramientas de escritorio Wayland y runtime core (`key-cli` con todas las opciones) **a nivel de sistema, sin modificar la configuración de ningún usuario**.
> * **Paso 2:** Creación y aprovisionamiento del usuario dedicado para el entorno Niri y configuración de SDDM. *(Ver [playbook-paso-2-usuario-niri-y-sddm.md](file:///home/gracie/Q/playbook-paso-2-usuario-niri-y-sddm.md))*
> * **Paso 3:** Configuración del entorno de escritorio, Clavis Shell / dotfiles y servicios de sesión para el nuevo usuario.

---

## 🎯 Objetivo del Paso 1

Instalar en un sistema Linux (optimizado para **Arch Linux / Omarchy**) el compositor **`niri-edge`** desde su repositorio fork y el runtime **`key-cli`** desde su rama más reciente (`HEAD`), junto con **todas las integraciones opcionales y permisos de hardware** (acceso a eventos de teclado evdev, lectura de energía de CPU en tiempo real vía RAPL, historial de portapapeles y captura de pantalla/audio), garantizando que **no se toque ningún archivo de configuración en `~/.config/`** del usuario actual.

---

## 📦 Repositorios y Fuentes

| Componente | Repositorio Oficial | Rama / Versión | Descripción |
|---|---|---|---|
| **niri-edge** | `https://github.com/StatIndet/niri-edge.git` | `main` (HEAD) | Fork de Niri con soporte nativo de minimización de ventanas y mejoras de borde. |
| **key-cli** | `https://github.com/StatIndet/key-cli.git` | `main` (HEAD) | CLI complementario para Clavis Shell (IPC, métricas de hardware nativas, grabación, portapapeles y teclado). |

---

## 🛠️ Requisitos Previos y Dependencias del Sistema

Antes de compilar, se deben instalar los paquetes base mediante `pacman`:

```bash
sudo pacman -S --needed --noconfirm \
  rust cargo clang cmake ninja qt6-base base-devel \
  waybar fuzzel alacritty mako swaybg swaylock xwayland-satellite \
  xdg-desktop-portal-gnome polkit-gnome wl-clipboard brightnessctl playerctl \
  cliphist libqalculate python-evdev python-pyudev gpu-screen-recorder slurp ffmpeg pactl
```

---

## 🔧 Proceso de Instalación Paso a Paso

### 1. Compilación e Instalación de `niri-edge`

```bash
# 1. Clonar o acceder al repositorio local
git clone https://github.com/StatIndet/niri-edge.git /ruta/a/niri-edge
cd /ruta/a/niri-edge
git checkout main
git pull

# 2. Compilar en modo release
cargo build --release

# 3. Desplegar binarios en el sistema
sudo install -Dm755 target/release/niri /usr/local/bin/niri
sudo ln -sf /usr/local/bin/niri /usr/bin/niri

sudo install -Dm755 resources/niri-session /usr/local/bin/niri-session
sudo ln -sf /usr/local/bin/niri-session /usr/bin/niri-session

# 4. Registrar sesión Wayland para Display Managers (GDM/SDDM/Greetd)
sudo install -Dm644 resources/niri.desktop /usr/share/wayland-sessions/niri.desktop
sudo install -Dm644 resources/niri.desktop /usr/local/share/wayland-sessions/niri.desktop

# 5. Instalar configuración de portales XDG (requerido para capturas y compartir pantalla)
sudo install -Dm644 resources/niri-portals.conf /usr/share/xdg-desktop-portal/niri-portals.conf
sudo install -Dm644 resources/niri-portals.conf /usr/local/share/xdg-desktop-portal/niri-portals.conf

# 6. Instalar unidades systemd de usuario
sudo install -Dm644 resources/niri.service /usr/lib/systemd/user/niri.service
sudo install -Dm644 resources/niri-shutdown.target /usr/lib/systemd/user/niri-shutdown.target

# 7. Generar autocompletados de shell
sudo mkdir -p /usr/share/bash-completion/completions /usr/share/zsh/site-functions /usr/share/fish/vendor_completions.d
target/release/niri completions bash | sudo tee /usr/share/bash-completion/completions/niri > /dev/null
target/release/niri completions zsh | sudo tee /usr/share/zsh/site-functions/_niri > /dev/null
target/release/niri completions fish | sudo tee /usr/share/fish/vendor_completions.d/niri.fish > /dev/null
sudo chmod 644 /usr/share/bash-completion/completions/niri /usr/share/zsh/site-functions/_niri /usr/share/fish/vendor_completions.d/niri.fish

# 8. Recargar systemd
sudo systemctl --system daemon-reload
```

---

### 2. Compilación e Instalación de `key-cli` y Opciones Completas

`key-cli` incluye un instalador autónomo (`scripts/install.sh`) que construye un entorno virtual aislado (`/usr/local/lib/key-cli/venv/`), compila el monitor nativo en C++ (`key-sysmon`) y el lector de energía (`key-cpu-power`).

```bash
# 1. Clonar o acceder al repositorio local
git clone https://github.com/StatIndet/key-cli.git /ruta/a/key-cli
cd /ruta/a/key-cli
git checkout main
git pull

# 2. Ejecutar la instalación base (como usuario normal, pedirá sudo solo para desplegar)
./scripts/install.sh

# 3. HABILITACIÓN OPCIONAL 1: Acceso a eventos de teclado (Caps Lock / Num Lock)
# Instala la regla udev /etc/udev/rules.d/71-clavis-keyboard-leds.rules y recarga udev
./scripts/install.sh --keyboard enable --acknowledge-keyboard-access --apply

# 4. HABILITACIÓN OPCIONAL 2: Acceso a métricas de energía de CPU (RAPL)
# Permite a key-cpu-power leer /sys/class/powercap sin requerir permisos de root
sudo setcap 'cap_dac_read_search=ep' /usr/local/libexec/key-cli/key-cpu-power
```

---

## 🔍 Verificación y Pruebas de Calidad

Cualquier agente debe validar que la instalación fue exitosa ejecutando los siguientes diagnósticos:

1. **Validar versión y sintaxis de niri:**
   ```bash
   niri --version
   # Debe retornar algo similar a: niri 26.04 (v26.04-...)
   niri validate --config /ruta/a/niri-edge/resources/default-config.kdl
   # Debe indicar: "niri: config is valid"
   ```

2. **Diagnóstico integral de `key-cli`:**
   ```bash
   key doctor --json
   ```
   **Criterio de éxito estricto:**
   * `"ok": true`
   * `"missing": []`
   * Todas las características (`shell`, `ipc`, `record`, `audio`, `clipboard-list`, `calculator`, `keyboard`, etc.) deben figurar como `true`.

3. **Verificar acceso a eventos de teclado:**
   ```bash
   key keyboard status --format json
   ```
   * Debe responder con `"available": true` y `"error": null`.

4. **Verificar lectura de consumo de CPU en tiempo real:**
   ```bash
   key sysmon cpu --format json
   ```
   * En el objeto `"cpu"`, el campo `"powerWatts"` debe tener un número positivo real (no null).

5. **Verificar aislamiento de usuario:**
   ```bash
   test ! -d ~/.config/niri && test ! -d ~/.config/key && echo "✅ Aislamiento respetado"
   ```
