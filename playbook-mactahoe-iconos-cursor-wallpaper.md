# Playbook: Iconos MacTahoe, cursor MacTahoe y wallpaper en Hyprland (Omarchy) y niri (Clavis)

Guía paso a paso para ejecutar a mano o pasársela a un agente de IA (Cursor, Claude Code, Aider, etc.) en un equipo con **Omarchy / Arch Linux + Hyprland** y un segundo usuario que usa **niri + Clavis** (Quickshell).

---

## 🎯 Objetivo

1. Instalar **de forma global** (para todos los usuarios) el tema de iconos y el tema de cursor MacTahoe.
2. Activarlos en **Hyprland** (usuario principal) y en **niri** (usuario `niri`).
3. Poner el wallpaper de [Wallhaven 5y3571](https://wallhaven.cc/w/5y3571) en ambos entornos.

Valores elegidos (el sistema usa modo oscuro):

| Elemento | Valor |
|---|---|
| Tema de iconos | `MacTahoe-dark` |
| Tema de cursor | `MacTahoe-dark-cursors` |
| Tamaño de cursor | `24` |
| Wallpaper | `wallhaven-5y3571.jpg` (3840×1634) |

---

## 📁 Otros documentos en `~/Q/`

| Archivo | Contenido |
|---|---|
| `playbook-paso-1/2/3-*.md` | instalación de niri-edge + key-cli, usuario `niri` + SDDM, y la shell Clavis |
| `keybingings_niri.txt` | atajos de teclado de niri (los 12 de Clavis al principio) |
| `niri_showcase_video.txt` + `niri_showcase_unixporn_720p.mp4` | video del showcase del autor y comandos exactos de yt-dlp |
| `icons_theme_wallpaper.txt` | enlaces a los repos de MacTahoe y al wallpaper |
| `playbook-niri-dotfiles-dotbare.md` | dotfiles de `niri` con dotbare, publicados en [robert-flo/niri-dotfiles](https://github.com/robert-flo/niri-dotfiles) |
| `playbook-prueba-instalacion-sura.md` | prueba de instalación limpia de `niri-dotfiles` en el usuario `sura` y conclusiones |
| `playbook-snapshot-home-btrfs.md` | snapshot Btrfs de `/home` (estado tras este playbook), interno + disco externo, y cómo restaurar `/home/niri` |

Este playbook cubre: iconos, cursor, wallpaper y GTK (secciones 1–5); y en la PARTE 2 la réplica del escritorio del autor en niri (secciones 6–11).

---

## 🧭 Contexto del sistema

- Usuario principal: sesión **Hyprland** de Omarchy (config en Lua, `~/.config/hypr/*.lua`).
- Usuario secundario: `niri` (uid 1001), sesión **niri** con la shell **Clavis** (`/etc/xdg/quickshell/clavis`, CLI `key`).
- El usuario principal tiene `sudo` sin contraseña y puede actuar como `niri` con `sudo -u niri` (no hace falta `su - niri`).
- Para `gsettings` del usuario `niri` hay que apuntar a su bus D-Bus: `DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/1001/bus` (solo existe si ese usuario tiene sesión de systemd activa).

---

## 1️⃣ Instalar iconos y cursores de forma global

El repositorio pesa bastante y el `git clone` puede tardar varios minutos.

```bash
cd /tmp
git clone --depth 1 https://github.com/vinceliuice/MacTahoe-icon-theme.git
cd MacTahoe-icon-theme

# Iconos -> /usr/share/icons/MacTahoe, MacTahoe-light, MacTahoe-dark (variante de color por defecto: blue)
sudo ./install.sh -d /usr/share/icons

# Cursores -> /usr/share/icons/MacTahoe-cursors y MacTahoe-dark-cursors
# (como root, el script instala en /usr/share/icons automáticamente)
cd cursors
sudo ./install.sh
```

Verificación:

```bash
ls -d /usr/share/icons/MacTahoe*
# MacTahoe  MacTahoe-dark  MacTahoe-dark-cursors  MacTahoe-light  MacTahoe-cursors
```

Opciones del instalador de iconos (por si se quiere otra variante): `-t purple|red|green|nord|all`, `-b` (iconos de panel más gruesos).

---

## 2️⃣ Activar en Hyprland (usuario principal, Omarchy)

Omarchy **no gestiona** el tema de iconos ni el tema de cursor (solo fija `XCURSOR_SIZE`/`HYPRCURSOR_SIZE` en `/usr/share/omarchy/default/hypr/envs.lua`). Nunca editar `/usr/share/omarchy/`.

### 2.1 Iconos y cursor para apps GTK

```bash
gsettings set org.gnome.desktop.interface icon-theme 'MacTahoe-dark'
gsettings set org.gnome.desktop.interface cursor-theme 'MacTahoe-dark-cursors'
gsettings set org.gnome.desktop.interface cursor-size 24
```

### 2.2 Cursor en Hyprland (persistente)

Hacer backup y añadir al **final** de `~/.config/hypr/hyprland.lua`:

```bash
cp ~/.config/hypr/hyprland.lua ~/.config/hypr/hyprland.lua.bak.$(date +%s)
```

```lua
hl.env("XCURSOR_THEME", "MacTahoe-dark-cursors")
hl.env("XCURSOR_SIZE", "24")
```

Aplicar y validar:

```bash
hyprctl reload
hyprctl configerrors          # debe salir vacío
hyprctl setcursor MacTahoe-dark-cursors 24   # aplica en caliente
```

### 2.3 Aplicar en la sesión actual sin cerrar sesión

`hl.env` en `hyprland.lua` no llega al entorno de systemd/D-Bus de la sesión ya iniciada (Omarchy usa UWSM), así que las apps lanzadas después (Electron, Qt, XWayland) seguirían con el cursor anterior hasta el próximo login. Para empujarlo ya:

```bash
systemctl --user set-environment XCURSOR_THEME=MacTahoe-dark-cursors XCURSOR_SIZE=24
dbus-update-activation-environment --systemd XCURSOR_THEME=MacTahoe-dark-cursors XCURSOR_SIZE=24
systemctl --user show-environment | grep XCURSOR   # debe mostrar XCURSOR_THEME=MacTahoe-dark-cursors
```

Las apps ya abiertas necesitan reiniciarse.

### 2.4 Cursor por defecto para apps XWayland / que no leen gsettings

```bash
mkdir -p ~/.icons/default
printf '[Icon Theme]\nInherits=MacTahoe-dark-cursors\n' > ~/.icons/default/index.theme
```

---

## 3️⃣ Activar en niri (usuario `niri`, Clavis)

Estado original encontrado: `Adwaita` en iconos y cursor, tamaño 24. Clavis solo fija el tamaño en `~/.config/niri/clavis/cursor.kdl`.

```bash
sudo -u niri env DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/1001/bus bash -c '
gsettings set org.gnome.desktop.interface icon-theme "MacTahoe-dark"
gsettings set org.gnome.desktop.interface cursor-theme "MacTahoe-dark-cursors"
gsettings set org.gnome.desktop.interface cursor-size 24

mkdir -p ~/.icons/default
printf "[Icon Theme]\nInherits=MacTahoe-dark-cursors\n" > ~/.icons/default/index.theme

# Archivo propio (NO editar clavis/cursor.kdl: lo gestiona Clavis y lo sobrescribe)
printf "cursor {\n    xcursor-theme \"MacTahoe-dark-cursors\"\n    xcursor-size 24\n}\n" > ~/.config/niri/cursor-theme.kdl

# Incluirlo al final de config.kdl (después de los include de clavis/*, para que gane)
grep -q "cursor-theme.kdl" ~/.config/niri/config.kdl || printf "\ninclude \"cursor-theme.kdl\";\n" >> ~/.config/niri/config.kdl

niri validate -c ~/.config/niri/config.kdl
'
```

Resultado esperado de `niri validate`: `config is valid`.

> ⚠️ Antes de tocar `config.kdl`, comprobar que nadie lo tenga abierto en un editor (`pgrep -u niri -a`). Si estaba abierto (p. ej. en `micro`) y se guarda desde ahí, se pierde la línea `include "cursor-theme.kdl";`.

> Nota: como `cursor-theme.kdl` se incluye al final, el tamaño de cursor que se cambie desde Clavis quedará anulado por el `24` de este archivo.

---

## 4️⃣ Wallpaper

### 4.1 Descargar (ubicación global legible por ambos usuarios)

La URL real se obtiene de la API de Wallhaven:

```bash
curl -s https://wallhaven.cc/api/v1/w/5y3571 \
  | python -c 'import json,sys;d=json.load(sys.stdin)["data"];print(d["path"],d["resolution"])'
# https://w.wallhaven.cc/full/5y/wallhaven-5y3571.jpg 3840x1634

sudo mkdir -p /usr/share/backgrounds
sudo curl -sL -o /usr/share/backgrounds/wallhaven-5y3571.jpg https://w.wallhaven.cc/full/5y/wallhaven-5y3571.jpg
sudo chmod 644 /usr/share/backgrounds/wallhaven-5y3571.jpg
```

### 4.2 Hyprland (Omarchy)

Los fondos de usuario van en `~/.config/omarchy/backgrounds/<slug-del-tema>/`. Ver el tema actual con `omarchy theme current` (aquí: *Tokyo Night* → `tokyo-night`).

```bash
mkdir -p ~/.config/omarchy/backgrounds/tokyo-night
cp /usr/share/backgrounds/wallhaven-5y3571.jpg ~/.config/omarchy/backgrounds/tokyo-night/
omarchy theme bg set ~/.config/omarchy/backgrounds/tokyo-night/wallhaven-5y3571.jpg
omarchy theme bg current     # -> Wallhaven 5y3571
```

Así además entra en el ciclo de `omarchy theme bg next`.

### 4.3 niri (Clavis)

Clavis guarda su configuración en `~/.config/clavis/config.json` (clave `wallpaper.path`) y su carpeta de fondos por defecto es `~/.local/share/clavis/wallpapers`. Acepta un JSON **parcial**: al arrancar lo completa con los valores por defecto.

```bash
sudo -u niri bash -c '
mkdir -p ~/.local/share/clavis/wallpapers ~/.config/clavis
cp /usr/share/backgrounds/wallhaven-5y3571.jpg ~/.local/share/clavis/wallpapers/
cat > ~/.config/clavis/config.json <<EOF
{
  "wallpaper": {
    "folder": "/home/niri/.local/share/clavis/wallpapers",
    "path": "/home/niri/.local/share/clavis/wallpapers/wallhaven-5y3571.jpg"
  }
}
EOF
python -m json.tool ~/.config/clavis/config.json >/dev/null && echo JSON_OK
'
```

> ⚠️ Si `~/.config/clavis/config.json` **ya existe**, no sobrescribirlo: editar solo `wallpaper.path` (y `wallpaper.folder`) con `jq` o Python, o usar el IPC de abajo.

Si Clavis ya está corriendo en una sesión de niri, se puede aplicar en caliente **desde dentro de esa sesión**:

```bash
qs -c clavis ipc call wallpaper set /home/niri/.local/share/clavis/wallpapers/wallhaven-5y3571.jpg
```

También se puede elegir desde Clavis con `Mod + Alt + W` (Spotlight → fondos de pantalla).

---

## 5️⃣ Tema GTK MacTahoe (solo Hyprland, en pruebas)

Repo: https://github.com/vinceliuice/MacTahoe-gtk-theme

> ⚠️ **Trampa del instalador en Arch:** si falta `sassc`, `glib-compile-resources` o `xmllint`, `install.sh` ejecuta por su cuenta `sudo pacman -Syyu --noconfirm ...` = **actualización completa del sistema sin preguntar**. Instalar antes las dependencias para que no lo haga:
>
> ```bash
> which sassc glib-compile-resources xmllint   # los tres deben existir
> sudo pacman -S --needed sassc                # (se eligió instalar solo sassc, sin -Syu)
> ```
>
> Si ya arrancó el `-Syyu`: revisar `tail /var/log/pacman.log`. Si **no** aparece `transaction started` (solo está descargando), se puede detener con `sudo kill -INT <pid de pacman>` sin riesgo. Si ya empezó la transacción, **no** interrumpir.

### 5.1 Instalar global

```bash
cd /tmp && git clone --depth 1 https://github.com/vinceliuice/MacTahoe-gtk-theme.git
cd MacTahoe-gtk-theme
sudo ./install.sh -d /usr/share/themes
# -> /usr/share/themes/MacTahoe-{Dark,Light}[-solid][-hdpi|-xhdpi]
# Efecto secundario: instala también /usr/bin/gnome-theme-switcher y su .desktop
```

### 5.2 Activar en Hyprland (usuario principal)

```bash
# GTK3
gsettings set org.gnome.desktop.interface gtk-theme 'MacTahoe-Dark'

# GTK4 / libadwaita: se instala en ~/.config/gtk-4.0 y NO se puede ejecutar con sudo.
# -d a una carpeta temporal para no duplicar el tema en ~/.themes
./install.sh -l -c dark -d /tmp/mactahoe-discard
rm -rf /tmp/mactahoe-discard
rm -f ~/.local/bin/gnome-theme-switcher ~/.local/share/applications/org.gnome.GTK4ThemeSwitcher.desktop
```

Omarchy solo fija `gtk-theme` en el primer arranque (`install/user/first-run/gnome-theme.sh`: `Adwaita-dark`), así que no lo sobrescribe al cambiar de tema.

Revertir en Hyprland:

```bash
gsettings set org.gnome.desktop.interface gtk-theme 'Adwaita-dark'
./install.sh -l -r     # quita los archivos de ~/.config/gtk-4.0 (o: rm -rf ~/.config/gtk-4.0)
```

### 5.3 📝 Nota: niri NO tenía el tema GTK configurado

> **Actualización:** ya se activó en niri en la sección **9️⃣** (botones tipo macOS). Lo de abajo es el estado original.

Investigado en el usuario `niri` (sin cambiar nada):

| Ajuste | Valor actual en niri |
|---|---|
| `gtk-theme` | `'Adwaita'` |
| `color-scheme` | `'default'` (ni siquiera modo oscuro) |
| `~/.config/gtk-3.0/settings.ini` | no existe |
| `~/.config/gtk-4.0/` | no existe |
| `~/.themes`, `~/.local/share/themes` | no existen |

Para activarlo en niri cuando se decida (el tema ya está instalado globalmente):

```bash
sudo -u niri env DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/1001/bus bash -c '
gsettings set org.gnome.desktop.interface gtk-theme "MacTahoe-Dark"
gsettings set org.gnome.desktop.interface color-scheme "prefer-dark"
'
# libadwaita: ejecutar como el usuario niri (no con sudo a secas, necesita su $HOME):
sudo -u niri -H bash -c 'cd /tmp/MacTahoe-gtk-theme && ./install.sh -l -c dark -d /tmp/mactahoe-discard-niri && rm -rf /tmp/mactahoe-discard-niri'
```

> Comprobado en el código de Clavis (`/etc/xdg/quickshell/clavis`): **no toca `gtk-theme`**, pero **sí cambia `color-scheme`** (`scripts/theme/set_system_color_scheme.sh`) al alternar modo claro/oscuro. Con libadwaita instalado (`gtk.css` → `gtk-Dark.css` fijo), las apps GTK4 seguirán oscuras aunque Clavis pase a modo claro.

---

## ✅ Verificación final

```bash
# Hyprland (usuario principal)
gsettings get org.gnome.desktop.interface icon-theme     # 'MacTahoe-dark'
gsettings get org.gnome.desktop.interface cursor-theme   # 'MacTahoe-dark-cursors'
hyprctl configerrors                                     # vacío
omarchy theme bg current                                 # Wallhaven 5y3571

# niri
sudo -u niri env DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/1001/bus \
  gsettings get org.gnome.desktop.interface icon-theme   # 'MacTahoe-dark'
sudo -u niri niri validate -c /home/niri/.config/niri/config.kdl   # config is valid
sudo -u niri cat /home/niri/.config/clavis/config.json
```

Las apps ya abiertas no cambian de iconos/cursor hasta reiniciarlas. En niri conviene cerrar y volver a abrir sesión.

---

## ↩️ Revertir

```bash
# Hyprland
gsettings set org.gnome.desktop.interface icon-theme 'Yaru-blue'
gsettings set org.gnome.desktop.interface cursor-theme 'default'
# quitar las dos líneas hl.env(...) del final de ~/.config/hypr/hyprland.lua (o restaurar el .bak) y `hyprctl reload`
rm ~/.icons/default/index.theme
omarchy theme bg next

# niri
sudo -u niri env DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/1001/bus bash -c '
gsettings set org.gnome.desktop.interface icon-theme Adwaita
gsettings set org.gnome.desktop.interface cursor-theme Adwaita
rm -f ~/.icons/default/index.theme ~/.config/niri/cursor-theme.kdl
sed -i "/include \"cursor-theme.kdl\";/d" ~/.config/niri/config.kdl
'

# Desinstalar temas globales
sudo rm -rf /usr/share/icons/MacTahoe /usr/share/icons/MacTahoe-light /usr/share/icons/MacTahoe-dark \
            /usr/share/icons/MacTahoe-cursors /usr/share/icons/MacTahoe-dark-cursors

# Tema GTK (ver también 5.2 para revertir en Hyprland)
sudo rm -rf /usr/share/themes/MacTahoe-*
sudo rm -f /usr/bin/gnome-theme-switcher /usr/share/applications/org.gnome.GTK4ThemeSwitcher.desktop
```

---
---

# PARTE 2 — Replicar el escritorio del autor de Clavis en niri

Contexto: el autor de Clavis / niri-edge ([StatIndet](https://github.com/StatIndet)) explicó en Reddit que lo que se ve en su video no es un solo tema global, sino varias piezas: Clavis (Quickshell + C++), su fork de niri con minimizar (efectos Scale y Genie), mpv en Lua basado en uosc, temas CSS de Obsidian y Thunderbird, botones tipo macOS **tematizados por app**, y recursos MacTahoe/LiquidBird.

Ya estaba instalado (ver `playbook-paso-1-niri-edge-y-key-cli.md`): `niri-edge`, `key-cli`, Clavis en `/etc/xdg/quickshell/clavis` y sus módulos C++ en `/usr/lib/qt6/qml/Clavis/*`.

Orden de trabajo (una tarea a la vez, todo en el usuario `niri`, verificado con `sudo -u niri`):

| # | Pieza | Repo del autor | Estado |
|---|---|---|---|
| 6 | mpv Cupertino (uosc) | [StatIndet/mpv](https://github.com/StatIndet/mpv) | ✅ hecho |
| 7 | Tema Obsidian | [StatIndet/obsidian-violet](https://github.com/StatIndet/obsidian-violet) | ✅ hecho |
| 8 | Tema Thunderbird | [StatIndet/liquidbird](https://github.com/StatIndet/liquidbird) | ✅ hecho |
| 9 | Botones tipo macOS por app | temas de cada app + MacTahoe GTK | ✅ hecho |
| 10 | Extras | `clavis-fish-theme`, `fcitx5-matugen-theme`, `wallpaper` | ✅ hecho |
| 11 | Piezas del zip `dotfiles` | [StatIndet/dotfiles](https://github.com/StatIndet/dotfiles) | ✅ animaciones, focus ring, Qt→GTK · ❌ fonts.conf (no aplica) |

---

## 6️⃣ mpv "Cupertino" (uosc 5.13.0 personalizado) — usuario `niri`

Qué aporta: barra de título transparente con botones rojo/amarillo/verde que aparecen al acercar el ratón, línea de tiempo por capítulos, miniaturas (thumbfast), animaciones de avance/retroceso y velocidad 2× manteniendo pulsada la flecha derecha o el botón derecho.

Requisitos (ya presentes en este equipo): `mpv` ≥ 0.40 (hay 0.41), `noto-fonts`, `yt-dlp`. En Intel además `vulkan-intel` e `intel-media-driver`.

> ⚠️ El repo modifica el código de uosc directamente: **no** usar el instalador oficial de uosc encima, sobrescribiría la personalización.

### 6.1 Instalar la configuración

`~/.config/mpv` de `niri` estaba vacío, no hizo falta backup (si no está vacío: `cp -a ~/.config/mpv ~/.config/mpv.bak.$(date +%s)`).

```bash
cd /tmp && git clone --depth 1 https://github.com/StatIndet/mpv.git statindet-mpv

sudo -u niri bash -c '
cd /tmp/statindet-mpv
cp -r fonts input.conf mpv.conf script-opts scripts ~/.config/mpv/
mkdir -p ~/Pictures/mpv          # screenshot-directory del mpv.conf
'
```

### 6.2 Adaptar a la GPU (el repo viene para NVIDIA)

El `mpv.conf` trae `hwdec=nvdec` (NVIDIA). Este equipo tiene **Intel HD 530** (`lspci | grep -i vga`):

```bash
sudo -u niri sed -i 's/^hwdec=nvdec/hwdec=vaapi/' /home/niri/.config/mpv/mpv.conf
```

- Se mantiene `vo=gpu-next` + `gpu-api=vulkan` (hay `vulkan-intel`).
- `hwdec=auto-safe` también funciona, pero primero intenta Vulkan Video y CUDA (no soportados aquí) y llena el log de errores; `vaapi` va directo.
- Otras GPUs: AMD → `vaapi`; NVIDIA → dejar `nvdec`.

### 6.3 Ayudante de uosc (búsqueda/descarga de subtítulos)

No viene en el repo; se toma del release oficial 5.13.0 (la descarga desde GitHub puede tardar varios minutos):

```bash
cd /tmp
curl -sL -o uosc-5.13.0.zip https://github.com/tomasklaen/uosc/releases/download/5.13.0/uosc.zip
unzip -q uosc-5.13.0.zip "scripts/uosc/bin/ziggy-linux" -d uosc-rel
sudo -u niri install -Dm755 uosc-rel/scripts/uosc/bin/ziggy-linux \
  /home/niri/.config/mpv/scripts/uosc/bin/ziggy-linux
```

(Ejecutarlo sin argumentos da `panic: slice bounds out of range`: es normal, uosc lo llama con argumentos.)

### 6.4 Verificación (sin abrir ventana)

```bash
sudo -u niri env -u DISPLAY -u WAYLAND_DISPLAY \
  DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/1001/bus XDG_RUNTIME_DIR=/run/user/1001 \
  bash -c 'cd ~; timeout -s KILL 25 mpv --vo=null --ao=null --frames=60 --keep-open=no --msg-level=all=warn /ruta/a/video.mp4; echo exit=$?'
# esperado: exit=0 y sin errores de scripts
```

> ⚠️ **Trampas al probar desde otra sesión (no son fallos de la config):**
> - `keep-open=yes` del `mpv.conf` deja mpv esperando al final del vídeo → en pruebas pasar `--keep-open=no`.
> - `/etc/mpv/scripts/mpris.so` (paquete `mpv-mpris`) **se cuelga si no hay bus D-Bus de sesión**: con `sudo -u niri` a secas mpv no termina nunca. Pasar siempre `DBUS_SESSION_BUS_ADDRESS` y `XDG_RUNTIME_DIR` del usuario `niri`. Dentro de la sesión gráfica de niri no pasa.
> - Usar `timeout -s KILL`: mpv ignora el `SIGTERM` de `timeout` en ese estado.

Prueba visual pendiente dentro de la sesión de niri: abrir un vídeo y comprobar la barra con botones rojo/amarillo/verde y la línea de tiempo.

### 6.5 Atajos principales (del README)

| Tecla | Acción |
|---|---|
| Espacio / clic | Reproducir / pausa |
| ← / → (pulsación corta) | −5 s / +5 s |
| J / L | −10 s / +10 s |
| Mantener → o botón derecho | 2× temporal |
| Clic derecho corto | Menú |
| F | Pantalla completa |
| I | Estadísticas |

Ajustes: `~/.config/mpv/script-opts/uosc.conf` (interfaz) y `script-opts/hold-speed.conf` (velocidad).

### 6.6 Revertir

```bash
sudo -u niri bash -c 'rm -rf ~/.config/mpv/{fonts,input.conf,mpv.conf,script-opts,scripts}'
```

---

## 7️⃣ Tema de Obsidian "Violet" — usuario `niri`

Qué aporta: barra superior y lateral discretas alrededor de una "tarjeta" de escritura opaca, pestañas tipo píldora, menús redondeados y **botones de ventana estilo macOS** (recursos de MacTahoe GTK). El texto Markdown mantiene la tipografía por defecto de Obsidian. Solo CSS: sin plugins, npm ni JavaScript.

Requisitos: `obsidian` ≥ 1.13.7 (instalado: 1.13.7). Versión del tema usada: **v0.1.1** (la de la release = la del repo).

> Los temas de Obsidian son **por vault**. El usuario `niri` nunca había abierto Obsidian (no había `~/.config/obsidian/obsidian.json` ni ningún vault), así que se creó uno.

### 7.1 Crear el vault con Violet activado

```bash
cd /tmp && git clone --depth 1 https://github.com/StatIndet/obsidian-violet.git

sudo -u niri bash -c '
V=~/Documents/Notas
mkdir -p "$V/.obsidian/themes/Violet" "$V/.obsidian/snippets"
cp /tmp/obsidian-violet/manifest.json /tmp/obsidian-violet/theme.css "$V/.obsidian/themes/Violet/"
cp /tmp/obsidian-violet/integration/native-glass.css "$V/.obsidian/snippets/violet-native-glass.css"

# Tema activo en el vault (snippet de cristal instalado pero DESACTIVADO)
cat > "$V/.obsidian/appearance.json" <<JSON
{
  "cssTheme": "Violet",
  "theme": "obsidian",
  "enabledCssSnippets": []
}
JSON
printf "# Notas\n\nVault de prueba con el tema Violet.\n" > "$V/Bienvenida.md"
'
```

`"theme": "obsidian"` = modo oscuro (`"moonstone"` = claro). Violet tiene ambos.

### 7.2 Registrar el vault para que Obsidian lo abra

**Solo si `~/.config/obsidian/obsidian.json` NO existe** (si existe, añadir la entrada dentro de `"vaults"` sin borrar las demás):

```bash
sudo -u niri bash -c '
ID=$(head -c 8 /dev/urandom | od -An -tx1 | tr -d " \n")   # id de 16 hex
TS=$(date +%s%3N)
cat > ~/.config/obsidian/obsidian.json <<JSON
{
  "vaults": {
    "$ID": { "path": "/home/niri/Documents/Notas", "ts": $TS, "open": true }
  }
}
JSON
'
```

Alternativa manual: abrir Obsidian → "Open folder as vault" → `~/Documents/Notas` → **Ajustes → Apariencia → Temas → Violet**.

### 7.3 Efecto cristal (opcional, en dos partes)

El autor avisa: el cristal real necesita que la ventana Electron de Obsidian sea **transparente de verdad** *y* que niri difumine el fondo. El CSS solo no basta.

**Parte niri (aplicada):** regla de ventana del autor en un archivo propio (esquinas de 18 px, recorte a la geometría, blur solo donde la ventana exponga transparencia; sin transparencia solo se notan las esquinas redondeadas).

```bash
sudo -u niri bash -c '
cp ~/.config/niri/config.kdl ~/.config/niri/config.kdl.bak.$(date +%s)
cp /tmp/obsidian-violet/integration/niri.kdl ~/.config/niri/obsidian-violet.kdl
grep -q "obsidian-violet.kdl" ~/.config/niri/config.kdl || printf "\ninclude \"obsidian-violet.kdl\";\n" >> ~/.config/niri/config.kdl
niri validate -c ~/.config/niri/config.kdl     # -> config is valid
'
```

La regla apunta a `app-id="^md\.obsidian\.Obsidian$"` (Obsidian nativo en Wayland).

**Parte Obsidian (NO activada):** el fragmento `violet-native-glass.css` está copiado en `.obsidian/snippets/`. Activarlo solo tras comprobar que la ventana es transparente: **Ajustes → Apariencia → Fragmentos CSS → violet-native-glass**.

> Nota: `~/.config/obsidian/user-flags.conf` de `niri` trae `-disable-gpu` (venía así de antes). Sin GPU es poco probable que Electron entregue una ventana transparente; si se quiere probar el cristal, quitar esa línea primero.

### 7.4 Verificación

```bash
sudo -u niri bash -c '
python -m json.tool ~/.config/obsidian/obsidian.json
cat ~/Documents/Notas/.obsidian/appearance.json          # cssTheme: Violet
ls ~/Documents/Notas/.obsidian/themes/Violet              # manifest.json theme.css
niri validate -c ~/.config/niri/config.kdl
'
```

Prueba visual pendiente dentro de la sesión de niri: abrir Obsidian, debe abrir el vault "Notas" con Violet y botones rojo/amarillo/verde.

### 7.5 Revertir

```bash
sudo -u niri bash -c '
rm -rf ~/Documents/Notas/.obsidian/themes/Violet ~/Documents/Notas/.obsidian/snippets/violet-native-glass.css
sed -i "s/\"cssTheme\": \"Violet\"/\"cssTheme\": \"\"/" ~/Documents/Notas/.obsidian/appearance.json
rm -f ~/.config/niri/obsidian-violet.kdl
sed -i "/include \"obsidian-violet.kdl\";/d" ~/.config/niri/config.kdl
'
```

---

## 8️⃣ Thunderbird + tema "LiquidBird" — usuario `niri`

Qué aporta: tema "Liquid Glass" estilo macOS para Thunderbird (correo, calendario, contactos, redacción, ajustes), con **botones de ventana MacTahoe** (`linux/titlebuttons.css`), iconos Lucide y barra lateral translúcida que niri difumina. Solo CSS (`userChrome.css` / `userContent.css`).

> ⚠️ Thunderbird no da soporte oficial a `userChrome.css`; una actualización puede romper selectores. Probado por el autor con Thunderbird 156.0; aquí se instaló **155.0** (la del repo de Arch).

### 8.1 Instalar Thunderbird

Comprobar antes que no choca con actualizaciones pendientes (en Arch, instalar sin `-Syu` es una actualización parcial):

```bash
checkupdates                                   # aquí: solo aether, cursor-bin, google-chrome, edge, mise-bin
pacman -S --print --print-format "%n %v" thunderbird
# -> mailcap, dbus-glib, botan, thunderbird  (solo paquetes nuevos, no toca nada instalado)
sudo pacman -S --needed --noconfirm thunderbird
thunderbird --version                          # Mozilla Thunderbird 155.0
```

La descarga (~80 MB) tardó más de una hora con la conexión de este equipo.

### 8.2 Empaquetar LiquidBird desde el repo

La release publicada (v0.1.0) es anterior al port de Linux; el autor indica empaquetar desde el repo hasta que haya release Linux:

```bash
cd /tmp && git clone --depth 1 https://github.com/StatIndet/liquidbird.git
cd liquidbird
python3 scripts/package_release.py --output dist
(cd dist && sha256sum -c SHA256SUMS)          # LiquidBird-0.1.0.zip: OK
```

### 8.3 Crear el perfil e instalar el tema

`niri` nunca había abierto Thunderbird. Desde la 15x, Thunderbird guarda perfiles en **`~/.config/thunderbird/`** (no en `~/.thunderbird`).

> ⚠️ **Corrección (2026-10-08).** `--CreateProfile` + `Default=1` **no basta**. En el primer arranque real, Thunderbird creó su propio perfil `ekf5x1zc.default-release-1` y lo fijó en `[Install<hash>] Default=… Locked=1` (en `profiles.ini` e `installs.ini`). Esa sección manda sobre `Default=1`, así que el tema había quedado en un perfil (`e38d6ugu…`) que no se usaba. Solución aplicada: instalar `chrome/` y `user.js` en el perfil que figura en `[Install…] Default=` (`ekf5x1zc.default-release-1`), borrar el perfil vacío y dejar un solo `[Profile0]` con ese `Path` y `Default=1`. Verificado con `thunderbird --headless`: no se crea otro perfil y `prefs.js` del perfil activo contiene la pref de stylesheets. **En una instalación nueva**: arrancar Thunderbird una vez (`timeout -s KILL 20 thunderbird --headless`) y usar como `$P` el perfil de `[Install…] Default=`:
>
> ```bash
> P=~/.config/thunderbird/$(sed -n "s/^Default=//p" ~/.config/thunderbird/installs.ini | head -1)
> ```

```bash
# Crear perfil sin abrir ventana
sudo -u niri env -u DISPLAY -u WAYLAND_DISPLAY bash -c 'cd ~; thunderbird --headless --CreateProfile "default-release"'

sudo -u niri bash -c '
P=$(ls -d ~/.config/thunderbird/*.default-release)
grep -q "^Default=1" ~/.config/thunderbird/profiles.ini || sed -i "/^Path=.*default-release/a Default=1" ~/.config/thunderbird/profiles.ini

# Equivale a Config Editor -> toolkit.legacyUserProfileCustomizations.stylesheets = true
echo "user_pref(\"toolkit.legacyUserProfileCustomizations.stylesheets\", true);" >> "$P/user.js"

T=$(mktemp -d); unzip -q /tmp/liquidbird/dist/LiquidBird-0.1.0.zip -d "$T"
mkdir -p "$P/chrome"
cp -r "$T/chrome/"{liquidbird.css,liquidbird-content.css,Icons,linux,userChrome.css,userContent.css} "$P/chrome/"
[ -f "$P/chrome/custom.css" ] || cp "$T/chrome/custom.css.example" "$P/chrome/custom.css"
rm -rf "$T"
'
```

> Si el perfil **ya tenía** `chrome/userChrome.css` o `userContent.css`: no copiarlos encima; añadir al principio los `@import` (ver `docs/LINUX.md` del repo). Nunca sobrescribir un `custom.css` existente. Hacer backup de `chrome/` antes.

Personalizaciones propias solo en `chrome/custom.css` (p. ej. `:root { --lb-accent: AccentColor; }` para usar el color de acento del escritorio en vez del azul por defecto).

### 8.4 Blur de niri para la barra lateral

```bash
sudo -u niri bash -c '
cp ~/.config/niri/config.kdl ~/.config/niri/config.kdl.bak.$(date +%s)
cp /tmp/liquidbird/integration/niri.kdl ~/.config/niri/liquidbird.kdl
grep -q "liquidbird.kdl" ~/.config/niri/config.kdl || printf "\ninclude \"liquidbird.kdl\";\n" >> ~/.config/niri/config.kdl
niri validate -c ~/.config/niri/config.kdl     # -> config is valid
'
```

La regla apunta a `app-id="^org\.mozilla\.Thunderbird$"` (Wayland nativo). Si Thunderbird arrancara por X11, lanzarlo con `GDK_BACKEND=wayland MOZ_ENABLE_WAYLAND=1 thunderbird`.

### 8.5 Verificación

```bash
sudo -u niri bash -c '
P=~/.config/thunderbird/$(sed -n "s/^Default=//p" ~/.config/thunderbird/installs.ini | head -1)
cat ~/.config/thunderbird/profiles.ini      # un solo perfil, el mismo que [Install…] Default=
cat "$P/user.js"                            # pref de stylesheets = true
ls "$P/chrome"                              # custom.css Icons linux liquidbird*.css userChrome.css userContent.css
niri validate -c ~/.config/niri/config.kdl
'
```

> `prefs.js` solo se escribe al cerrar Thunderbird normalmente; `user.js` se aplica en cada arranque, así que no hace falta que la pref aparezca en `prefs.js`.
> Para probar sin ventana usar también `DBUS_SESSION_BUS_ADDRESS`/`XDG_RUNTIME_DIR` del usuario y `timeout -s KILL` (igual que con mpv).

Prueba visual pendiente dentro de la sesión de niri: abrir Thunderbird (pedirá configurar una cuenta), comprobar botones rojo/amarillo/verde y barra lateral translúcida. Para parecerse al README: densidad por defecto, vista de tarjetas y vista vertical.

Demo sin tocar el perfil real (perfil sintético aislado, sin conexión):

```bash
cd /tmp/liquidbird && python3 scripts/demo.py prepare && python3 scripts/demo.py launch
```

### 8.6 Revertir

```bash
sudo -u niri bash -c '
P=$(ls -d ~/.config/thunderbird/*.default-release)
rm -rf "$P/chrome"                          # (o restaurar el backup de chrome/)
sed -i "/legacyUserProfileCustomizations/d" "$P/user.js"
rm -f ~/.config/niri/liquidbird.kdl
sed -i "/include \"liquidbird.kdl\";/d" ~/.config/niri/config.kdl
'
sudo pacman -Rns thunderbird                 # si se quiere quitar Thunderbird
```

---

## 9️⃣ Botones tipo macOS (semáforo) — usuario `niri`

El autor aclara que los botones rojo/amarillo/verde **no vienen de un único tema GTK**: cada app los trae en su propio tema. Mapa de dónde salen:

| App | De dónde salen los botones | Estado |
|---|---|---|
| mpv | `scripts/uosc/assets/mactahoe/titlebutton-*.svg` (sección 6) | ✅ |
| Obsidian | tema Violet, recursos MacTahoe embebidos (sección 7) | ✅ |
| Thunderbird | LiquidBird `linux/titlebuttons.css` + `linux/titlebuttons/*.png` (sección 8) | ✅ |
| Apps GTK3/GTK4 (Archivos, Ajustes, etc.) | tema GTK **MacTahoe** (sección 5) | ✅ (esta sección) |
| Electron (Chrome, VS Code…), Qt, terminales sin CSD | no aplica: dibujan su propia barra o no tienen | — |

> Se revisó [StatIndet/dotfiles](https://github.com/StatIndet/dotfiles): está desactualizado (último commit 2026-05-24, anterior a Clavis; usa cursor Bibata, waybar, rofi…) y **no contiene temas de botones**. Su README remite al repo de Quickshell (= Clavis, ya instalado). No se usó. (Clonarlo es muy lento aquí; para inspeccionarlo basta la API: `curl -s https://api.github.com/repos/StatIndet/dotfiles/git/trees/HEAD?recursive=1`.)

### 9.1 Activar MacTahoe GTK en niri con botones a la izquierda (estilo macOS)

El tema ya está instalado globalmente (5.1).

```bash
sudo -u niri env DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/1001/bus bash -c '
gsettings set org.gnome.desktop.interface gtk-theme "MacTahoe-Dark"
gsettings set org.gnome.desktop.interface color-scheme "prefer-dark"
gsettings set org.gnome.desktop.wm.preferences button-layout "close,minimize,maximize:"

# Respaldo para apps que leen settings.ini en vez de gsettings
for v in 3.0 4.0; do
  mkdir -p ~/.config/gtk-$v; f=~/.config/gtk-$v/settings.ini
  [ -f $f ] && cp $f $f.bak.$(date +%s)
  printf "[Settings]\ngtk-theme-name=MacTahoe-Dark\ngtk-icon-theme-name=MacTahoe-dark\ngtk-cursor-theme-name=MacTahoe-dark-cursors\ngtk-cursor-theme-size=24\ngtk-application-prefer-dark-theme=1\ngtk-decoration-layout=close,minimize,maximize:\n" > $f
done
'
```

`close,minimize,maximize:` = botones a la izquierda en orden macOS (lo que va antes de `:` queda a la izquierda). Para dejarlos a la derecha: `":minimize,maximize,close"`.

### 9.2 GTK4 / libadwaita para niri

> ⚠️ `install.sh -l` falla en silencio ("Destination directory does not exist…" y termina) si el usuario **no puede escribir en la carpeta del repo**. Darle una copia propia:

```bash
cp -r /tmp/MacTahoe-gtk-theme /tmp/mactahoe-gtk-niri
sudo chown -R niri:niri /tmp/mactahoe-gtk-niri
sudo -u niri -H bash -c '
cd /tmp/mactahoe-gtk-niri && ./install.sh -l -c dark -d /tmp/mactahoe-discard-niri
rm -rf /tmp/mactahoe-discard-niri
rm -f ~/.local/bin/gnome-theme-switcher ~/.local/share/applications/org.gnome.GTK4ThemeSwitcher.desktop
'
sudo rm -rf /tmp/mactahoe-gtk-niri
```

El instalador respeta el `settings.ini` de `~/.config/gtk-4.0` (no lo sobrescribe).

### 9.3 Verificación

```bash
sudo -u niri env DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/1001/bus bash -c '
gsettings get org.gnome.desktop.interface gtk-theme          # MacTahoe-Dark
gsettings get org.gnome.desktop.wm.preferences button-layout # close,minimize,maximize:
readlink ~/.config/gtk-4.0/gtk.css                           # .../gtk-Dark.css
'
```

Prueba visual pendiente en la sesión de niri: abrir Nautilus (o cualquier app GTK) y ver los botones a la izquierda.

> Clavis cambia `color-scheme` al alternar claro/oscuro (no toca `gtk-theme`). Las apps GTK4 quedarán oscuras siempre por el enlace fijo `gtk.css → gtk-Dark.css`.

### 9.4 Revertir

```bash
sudo -u niri env DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/1001/bus bash -c '
gsettings set org.gnome.desktop.interface gtk-theme Adwaita
gsettings set org.gnome.desktop.interface color-scheme default
gsettings reset org.gnome.desktop.wm.preferences button-layout
rm -f ~/.config/gtk-3.0/settings.ini ~/.config/gtk-4.0/settings.ini
rm -rf ~/.config/gtk-4.0/{gtk.css,gtk-dark.css,gtk-Light.css,gtk-Dark.css,assets,windows-assets}
'
```

---

## 🔟 Extras — usuario `niri`

### 10.1 Cómo Clavis integra plantillas Matugen propias (base de 10.2 y 10.3)

Clavis regenera colores con Matugen cada vez que cambia el wallpaper. Además de sus plantillas internas (`/etc/xdg/quickshell/clavis/matugen/config.toml`: quickshell, btop, cava, kitty, yazi), lee plantillas del usuario en:

```text
~/.config/clavis/matugen/config.toml
```

> ⚠️ Es un **TOML limitado** (`scripts/theme/matugen_registry.jq`): solo `[templates.<id>]` con `input_path`, `output_path` y `post_hook`. **No** admite `index` ni la sección `[config]` (los README de Matugen los usan: hay que quitarlos o la plantilla queda inválida). IDs reservados: los internos de Clavis.

Las plantillas del usuario vienen **desactivadas**; se activan en `~/.config/clavis/config.json` → `theme.matugenTemplates.<id> = true` (o desde el centro de control de Clavis).

Validar el registro:

```bash
sudo -u niri bash -c 'cd /etc/xdg/quickshell/clavis/scripts && bash -c "source lib/matugen-registry.sh; matugen_registry_init; matugen_registry_list" | jq -c ".templates[] | select(.origin==\"user\") | {id, valid, error}"'
```

Generar ya (sin esperar a cambiar de wallpaper), solo las plantillas indicadas:

```bash
sudo -u niri env DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/1001/bus XDG_RUNTIME_DIR=/run/user/1001 bash -c '
bash /etc/xdg/quickshell/clavis/scripts/theme/generate_matugen_colors.sh \
  --image /home/niri/.local/share/clavis/wallpapers/wallhaven-5y3571.jpg --mode dark \
  --templates <id1>,<id2>,...'
# salida esperada: eventos JSON "core-ready" y "finished" con error ""
```

### 10.2 Tema de fcitx5 que sigue los colores del wallpaper ([fcitx5-matugen-theme](https://github.com/StatIndet/fcitx5-matugen-theme))

Requisitos (ya estaban): `fcitx5`, `matugen` (4.2.0), `librsvg` (`rsvg-convert`), `busctl` (systemd).

```bash
cd /tmp && git clone --depth 1 https://github.com/StatIndet/fcitx5-matugen-theme.git
# Ubicación estable dentro del usuario (las rutas quedan escritas en el registro)
sudo cp -r /tmp/fcitx5-matugen-theme /home/niri/.local/share/
sudo chown -R niri:niri /home/niri/.local/share/fcitx5-matugen-theme
sudo -u niri mkdir -p /home/niri/.local/share/fcitx5/themes/Matugen /home/niri/.config/clavis/matugen
```

Contenido añadido a `~/.config/clavis/matugen/config.toml` de `niri` (adaptado del README: sin `index`; `fcitx5_arrow` al final porque su `post_hook` recarga Fcitx5 cuando ya están todos los archivos):

```toml
[templates.fcitx5_theme]
input_path = "/home/niri/.local/share/fcitx5-matugen-theme/templates/theme.conf"
output_path = "~/.local/share/fcitx5/themes/Matugen/theme.conf"

[templates.fcitx5_panel]
input_path = "/home/niri/.local/share/fcitx5-matugen-theme/templates/panel.svg"
output_path = "~/.local/share/fcitx5/themes/Matugen/panel.svg"
post_hook = "bash \"/home/niri/.local/share/fcitx5-matugen-theme/scripts/render-panel.sh\" \"$HOME/.local/share/fcitx5/themes/Matugen/panel.svg\" \"$HOME/.local/share/fcitx5/themes/Matugen/panel.png\""

[templates.fcitx5_highlight]
input_path = "/home/niri/.local/share/fcitx5-matugen-theme/templates/highlight.svg"
output_path = "~/.local/share/fcitx5/themes/Matugen/highlight.svg"

[templates.fcitx5_radio]
input_path = "/home/niri/.local/share/fcitx5-matugen-theme/templates/radio.svg"
output_path = "~/.local/share/fcitx5/themes/Matugen/radio.svg"

[templates.fcitx5_arrow]
input_path = "/home/niri/.local/share/fcitx5-matugen-theme/templates/arrow.svg"
output_path = "~/.local/share/fcitx5/themes/Matugen/arrow.svg"
post_hook = "busctl --user call org.fcitx.Fcitx5 /controller org.fcitx.Fcitx.Controller1 ReloadAddonConfig s classicui"
```

Activar en Clavis, generar y seleccionar el tema en Fcitx5:

```bash
sudo -u niri python - <<'PY'
import json
p="/home/niri/.config/clavis/config.json"
d=json.load(open(p))
t=d.setdefault("theme",{}).setdefault("matugenTemplates",{})
for i in ["fcitx5_theme","fcitx5_panel","fcitx5_highlight","fcitx5_radio","fcitx5_arrow"]: t[i]=True
json.dump(d,open(p,"w"),indent=2)
PY
# (hacer backup de config.json antes)

# generar (ver 10.1) con --templates fcitx5_theme,fcitx5_panel,fcitx5_highlight,fcitx5_radio,fcitx5_arrow
# -> ~/.local/share/fcitx5/themes/Matugen/{theme.conf,panel.svg,panel.png,highlight.svg,radio.svg,arrow.svg}

# Elegir el tema (equivale a fcitx5-configtool -> Classic UI -> Theme = Matugen)
sudo -u niri bash -c 'mkdir -p ~/.config/fcitx5/conf && printf "Theme=Matugen\nDarkTheme=Matugen\n" > ~/.config/fcitx5/conf/classicui.conf'
```

> Nota: la sesión de niri **no arranca fcitx5** (no hay `spawn-at-startup "fcitx5"`). El tema está listo, pero solo se verá si se usa fcitx5 (método de entrada, p. ej. para chino/japonés). Para arrancarlo: añadir `spawn-at-startup "fcitx5" "-d"` en `config.kdl`.

### 10.3 Fish + prompt de Clavis ([clavis-fish-theme](https://github.com/StatIndet/clavis-fish-theme))

Prompt de dos líneas en "cápsulas" (Nerd Font + Powerline) con módulos de duración, ruta, Git, lenguaje y hora; colores Material 3 que cambian con el wallpaper.

Requisitos: Fish ≥ 4.1, Rust/Cargo, una Nerd Font (aquí: JetBrainsMono Nerd Font, ya instalada y usada por Alacritty), Git.

```bash
# Fish (comprobar que no arrastra actualizaciones: solo instala "fish 4.9.2-1")
pacman -S --print --print-format "%n %v" fish
sudo pacman -S --needed --noconfirm fish

# Compilar e instalar el binario para todos los usuarios
cd /tmp && git clone --depth 1 https://github.com/StatIndet/clavis-fish-theme.git
cd clavis-fish-theme && cargo build --release --locked
sudo install -Dm755 target/release/clavis-fish-theme /usr/local/bin/clavis-fish-theme
```

Configuración en `niri`:

```bash
sudo -u niri bash -c '
mkdir -p ~/.local/share/clavis-fish-theme ~/.config/clavis-fish-theme ~/.config/fish
cp /tmp/clavis-fish-theme/examples/matugen.conf ~/.local/share/clavis-fish-theme/matugen.conf

# Activar el prompt
grep -q "clavis-fish-theme init fish" ~/.config/fish/config.fish 2>/dev/null || \
  printf "if status is-interactive\n    clavis-fish-theme init fish | source\nend\n" >> ~/.config/fish/config.fish
'
```

Plantilla de colores registrada en Clavis (añadida a `~/.config/clavis/matugen/config.toml`) y activada como `theme.matugenTemplates.clavis_fish = true`:

```toml
[templates.clavis_fish]
input_path = "/home/niri/.local/share/clavis-fish-theme/matugen.conf"
output_path = "~/.config/clavis-fish-theme/colors.conf"
```

Generar con `--templates clavis_fish` (ver 10.1) → `~/.config/clavis-fish-theme/colors.conf`.

**Usar Fish en la terminal sin cambiar el shell de login** (el login de `niri` sigue en `/bin/bash`, los scripts no se ven afectados). En `~/.config/alacritty/alacritty.toml`, sección `[terminal]`:

```toml
[terminal]
osc52 = "CopyPaste"
shell = { program = "/usr/bin/fish" }
```

```bash
sudo -u niri python -c "import tomllib;tomllib.load(open('/home/niri/.config/alacritty/alacritty.toml','rb'));print('TOML_OK')"
```

(Alternativa más invasiva: `sudo chsh -s /usr/bin/fish niri`.)

Verificación (render del prompt sin terminal):

```bash
sudo -u niri bash -c 'cd ~; cat ~/.config/clavis-fish-theme/colors.conf; COLUMNS=100 fish -i -c fish_prompt | cat -v | head -3'
# debe mostrar secuencias de color 24-bit con los colores de colors.conf, la ruta "~" y la hora
```

Opcional: `~/.config/clavis-fish-theme/config.conf` para activar/desactivar módulos y márgenes (ejemplo en `examples/config.conf` del repo).

### 10.4 Wallpapers del autor en el selector de Clavis

Clavis busca wallpapers **recursivamente** (`find`) en `~/.local/share/clavis/wallpapers`, así que cada origen va en su propia subcarpeta:

| Origen | Destino (usuario `niri`) | Estado |
|---|---|---|
| carpeta `wallpaper/` del zip `dotfiles-master` (15 imágenes, ~45 MB) | `~/.local/share/clavis/wallpapers/statindet-dotfiles/` | ✅ copiado |
| repo [StatIndet/wallpaper](https://github.com/StatIndet/wallpaper) (35 imágenes; copia local ya clonada en `/run/media/gracie/ALMACEN_BTRFS/gracie/2026-10-07-StatIndet-wallpaper`, commit `79dd5cb`) | `~/.local/share/clavis/wallpapers/statindet/` | ✅ 20 copiadas (las otras 15 son idénticas a las del zip y se omitieron) |

```bash
# zip de dotfiles (descargado en ~/Downloads/dotfiles-master)
D=/home/niri/.local/share/clavis/wallpapers/statindet-dotfiles
sudo mkdir -p $D && sudo cp ~/Downloads/dotfiles-master/wallpaper/* $D/
sudo bash -c "chown -R niri:niri $D && chmod 644 $D/*"   # el glob debe expandirse como root

# repo wallpaper: usar la copia local (clonarlo con la red lenta tarda >1 h)
#   si no existe: git clone --depth 1 https://github.com/StatIndet/wallpaper.git
P="/run/media/gracie/ALMACEN_BTRFS/gracie/2026-10-07-StatIndet-wallpaper"
D=/home/niri/.local/share/clavis/wallpapers/statindet
sudo mkdir -p $D
# omitir las imágenes que ya vinieron en el zip (mismo md5) para no duplicarlas en el selector
sudo bash -c "md5sum /home/niri/.local/share/clavis/wallpapers/statindet-dotfiles/* | cut -d' ' -f1" > /tmp/dot.md5
find "$P" -path '*/.git' -prune -o -type f \( -iname '*.jpg' -o -iname '*.jpeg' -o -iname '*.png' -o -iname '*.webp' \) -print0 |
  while IFS= read -r -d '' f; do grep -qx "$(md5sum "$f" | cut -d' ' -f1)" /tmp/dot.md5 || sudo cp "$f" $D/; done
rm /tmp/dot.md5
sudo bash -c "chown -R niri:niri $D && chmod 644 $D/*"
```

Verificación:

```bash
sudo -u niri find /home/niri/.local/share/clavis/wallpapers -type f \( -iname '*.jpg' -o -iname '*.jpeg' -o -iname '*.png' -o -iname '*.webp' \) | wc -l
```

### 10.5 Revertir extras

```bash
sudo -u niri bash -c '
rm -f ~/.config/clavis/matugen/config.toml              # quita fcitx5 y clavis_fish del registro de Clavis
python - <<PY
import json; p="/home/niri/.config/clavis/config.json"; d=json.load(open(p))
for i in ["fcitx5_theme","fcitx5_panel","fcitx5_highlight","fcitx5_radio","fcitx5_arrow","clavis_fish"]: d.get("theme",{}).get("matugenTemplates",{}).pop(i,None)
json.dump(d,open(p,"w"),indent=2)
PY
rm -rf ~/.local/share/fcitx5-matugen-theme ~/.local/share/fcitx5/themes/Matugen ~/.config/fcitx5/conf/classicui.conf
sed -i "/clavis-fish-theme init fish/,+1d; /^if status is-interactive$/d" ~/.config/fish/config.fish
sed -i "/^shell = { program = \"\/usr\/bin\/fish\" }/d" ~/.config/alacritty/alacritty.toml
rm -rf ~/.local/share/clavis-fish-theme ~/.config/clavis-fish-theme
rm -rf ~/.local/share/clavis/wallpapers/statindet-dotfiles ~/.local/share/clavis/wallpapers/statindet
'
sudo rm -f /usr/local/bin/clavis-fish-theme
sudo pacman -Rns fish
```

---

## 1️⃣1️⃣ Piezas del repo `dotfiles` del autor — usuario `niri`

Fuente: [StatIndet/dotfiles](https://github.com/StatIndet/dotfiles), descargado como zip en `~/Downloads/dotfiles-master`. **Es su configuración anterior a Clavis** (waybar/matugen sueltos, etc.), así que solo se toma lo que no choca con lo que Clavis ya gestiona. Los archivos `~/.config/niri/clavis/*.kdl` son "Managed by Clavis": no se editan. Todo lo extra va en archivos propios incluidos al final de `config.kdl`.

| Pieza del zip | Decisión |
|---|---|
| `niri/animations.kdl` | ✅ aplicada (11.1) |
| `matugen/templates/niri-colors.kdl` (focus ring con degradado) | ✅ aplicada vía registro Matugen de Clavis (11.2) |
| `environment.d/envvars.conf` → `QT_QPA_PLATFORMTHEME=gtk3` | ✅ aplicada (11.3) |
| `wallpaper/` | ✅ copiada (10.4) |
| `fontconfig/fonts.conf` (LXGW WenKai) | ❌ no se aplica, ver investigación (11.4) |
| waybar, binds, startup, output, etc. | ❌ reemplazado por Clavis / config actual |

### 11.1 Animaciones (spring + window-open con rebote)

```bash
sudo install -o niri -g niri -m 644 ~/Downloads/dotfiles-master/niri/animations.kdl /home/niri/.config/niri/statindet-animations.kdl
sudo -u niri bash -c 'printf "\ninclude \"statindet-animations.kdl\";\n" >> ~/.config/niri/config.kdl
niri validate -c ~/.config/niri/config.kdl'
```

Contenido: animaciones `spring` para workspaces/vistas/movimiento, `window-open` 500 ms `cubic-bezier 0.05 0.9 0.1 1.05` (pequeño rebote), `window-close` 150 ms. Como va incluido al final, sobrescribe los valores anteriores del bloque `animations {}`.

### 11.2 Focus ring con degradado que sigue al wallpaper

El zip trae una plantilla Matugen (`matugen/templates/niri-colors.kdl`) que genera un `focus-ring` con `active-gradient` primary→tertiary. Se registra como plantilla de usuario de Clavis (mismo mecanismo que 10.1):

```bash
sudo -u niri mkdir -p /home/niri/.local/share/statindet-niri
sudo install -o niri -g niri -m 644 ~/Downloads/dotfiles-master/matugen/templates/niri-colors.kdl /home/niri/.local/share/statindet-niri/niri-colors.kdl
```

Añadir a `~/.config/clavis/matugen/config.toml` (usuario `niri`):

```toml
[templates.niri_focus_ring]
input_path = "/home/niri/.local/share/statindet-niri/niri-colors.kdl"
output_path = "~/.config/niri/statindet-colors.kdl"
```

Activar en `~/.config/clavis/config.json` → `theme.matugenTemplates.niri_focus_ring = true`, generar e incluir:

```bash
sudo -u niri bash -c '
WP=$(jq -r .wallpaper.path ~/.config/clavis/config.json)
/etc/xdg/quickshell/clavis/scripts/theme/generate_matugen_colors.sh --image "$WP" --mode dark --templates niri_focus_ring
printf "\ninclude \"statindet-colors.kdl\";\n" >> ~/.config/niri/config.kdl
cat ~/.config/niri/statindet-colors.kdl
niri validate -c ~/.config/niri/config.kdl'
```

Resultado con el wallpaper 5y3571: `active-gradient from="#85d2e7cc" to="#bfc4ebcc" angle=135`. Al cambiar el wallpaper desde Clavis se regenera solo y niri recarga el archivo automáticamente.

### 11.3 Apps Qt con tema GTK (`QT_QPA_PLATFORMTHEME=gtk3`)

Investigado antes de aplicar: Clavis no define ni depende de `QT_QPA_PLATFORMTHEME` (dibuja todo en QML con sus propios colores/fuentes), el plugin `/usr/lib/qt6/plugins/platformthemes/libqgtk3.so` está instalado y Omarchy usa este mismo valor en Hyprland. Se aplica igual que el autor, vía `environment.d` (lo carga systemd para toda la sesión):

```bash
sudo -u niri bash -c 'mkdir -p ~/.config/environment.d
printf "QT_QPA_PLATFORMTHEME=gtk3\n" > ~/.config/environment.d/qt-gtk3.conf
/usr/lib/systemd/user-environment-generators/30-systemd-environment-d-generator | grep QT_QPA'
# → QT_QPA_PLATFORMTHEME=gtk3   (efectivo al iniciar la próxima sesión de niri)
```

Efecto: apps Qt (diálogos de archivos, paleta, iconos) siguen el tema GTK MacTahoe-Dark y los iconos MacTahoe.

### 11.4 Fuentes LXGW WenKai — investigación (NO aplicado)

Hallazgos:

1. **Clavis ya usa LXGW WenKai por defecto, en su versión actual.** En `/etc/xdg/quickshell/clavis/Common/Fonts.qml`: `defaultUi: "LXGW WenKai GB Screen"`, `defaultMono: "JetBrainsMono Nerd Font"`; el reloj usa Google Sans Flex incluida en Clavis. `docs/dependencies.md` lista `ttf-lxgw-wenkai-screen` como "Default UI font". El paquete ya está instalado (`LXGW WenKai GB Screen`, `LXGW WenKai Mono GB Screen`, …). El usuario `niri` no tiene `theme.fonts` en `config.json`, así que Clavis ya está usando esa fuente.
2. **Clavis pide familias explícitas, no genéricas**, así que el `fonts.conf` del autor no le afecta (solo cae a `monospace` si falta JetBrainsMono Nerd Font).
3. **En Omarchy el `fonts.conf` del autor no hace nada.** `/etc/fonts/conf.d/50-omarchy.conf` se carga antes que `50-user.conf` y hace `mode="assign" binding="strong"` de `sans-serif`→Liberation Sans, `serif`→Liberation Serif, `monospace`→JetBrainsMono Nerd Font. Cuando llega la regla del usuario, la familia genérica ya fue reemplazada y su `<test>` no coincide. Simulado sin tocar nada:

   ```bash
   T=$(mktemp -d); mkdir -p $T/fontconfig; cp ~/Downloads/dotfiles-master/fontconfig/fonts.conf $T/fontconfig/
   for f in sans-serif serif monospace; do XDG_CONFIG_HOME=$T XDG_CACHE_HOME=$T/cache fc-match $f; done; rm -rf $T
   # → Liberation Sans / Liberation Serif / JetBrainsMono Nerd Font (igual que sin el archivo)
   ```

4. Para que tuviera efecto habría que adaptar la regla a las familias de Omarchy, y eso cambiaría la fuente **del resto de apps** (Chromium/Electron, Obsidian, Thunderbird, Qt), no la de Clavis. La parte `monospace` además sustituiría JetBrainsMono Nerd Font en apps que piden monospace genérico.

Conclusión: no hace falta aplicar nada para que Clavis se vea como el diseño del autor, porque ya usa LXGW WenKai. El `fonts.conf` antiguo no se copia.

### 11.5 Revertir piezas de `dotfiles`

```bash
sudo -u niri bash -c '
sed -i "/^include \"statindet-animations.kdl\";$/d; /^include \"statindet-colors.kdl\";$/d" ~/.config/niri/config.kdl
rm -f ~/.config/niri/statindet-animations.kdl ~/.config/niri/statindet-colors.kdl
rm -rf ~/.local/share/statindet-niri ~/.config/environment.d/qt-gtk3.conf
sed -i "/^\[templates.niri_focus_ring\]$/,/^output_path/d" ~/.config/clavis/matugen/config.toml
python - <<PY
import json; p="/home/niri/.config/clavis/config.json"; d=json.load(open(p))
d.get("theme",{}).get("matugenTemplates",{}).pop("niri_focus_ring",None)
json.dump(d,open(p,"w"),indent=2)
PY
niri validate -c ~/.config/niri/config.kdl'
```
