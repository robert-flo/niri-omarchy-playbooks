# Playbook: Prueba de instalación limpia de `niri-dotfiles` (usuario `sura`)

Objetivo: comprobar que [robert-flo/niri-dotfiles](https://github.com/robert-flo/niri-dotfiles) (commit `04eb5ce`) se instala bien con dotbare en un usuario nuevo y comparar el resultado con el snapshot de referencia `/home/.snapshots/home-2026-10-08-dotfiles/niri`.

Fecha: 2026-10-08.

---

## 1️⃣ Crear el usuario `sura` (mismos permisos que `niri`/`gracie`)

`niri` tiene `wheel,audio,input,video`, sudo sin contraseña (`/etc/sudoers.d/04_niri`) y linger; `gracie` tiene `wheel` y la misma regla de sudo. `sura` se crea igual:

```bash
sudo useradd -m -s /bin/bash -G wheel,audio,input,video sura
echo 'sura:<CONTRASEÑA>' | sudo chpasswd          # la contraseña la eligió el usuario; no se documenta aquí
printf 'sura ALL=(ALL) NOPASSWD: ALL\nDefaults:sura verifypw = any\n' | sudo tee /etc/sudoers.d/04_sura >/dev/null
sudo chmod 0440 /etc/sudoers.d/04_sura && sudo visudo -cf /etc/sudoers.d/04_sura
sudo loginctl enable-linger sura
```

Verificado: `uid=1002 groups=sura,wheel,audio,input,video`, `sudo -n true` OK. `/etc/skel` de Omarchy deja ya un `~/.config` (alacritty, hypr, nvim, obsidian, etc.).

## 2️⃣ Instalar con dotbare (exactamente como dice el README)

```bash
sudo -u sura env HOME=/home/sura DOTBARE_DIR=/home/sura/.cfg DOTBARE_TREE=/home/sura bash -c 'cd ~
dotbare finit -y -u https://github.com/robert-flo/niri-dotfiles.git </dev/null'
```

Resultado: `File checkout succeeded`, `Migration completed` (3 min con la red lenta). Conflictos con el skel de Omarchy respaldados automáticamente en `~/.local/share/dotbare/`: `.config/alacritty/alacritty.toml` y `.config/obsidian/user-flags.conf`. `git status` limpio.

## 3️⃣ Comparación con el snapshot

```bash
S=/home/.snapshots/home-2026-10-08-dotfiles/niri
sudo bash -c "cd /home/sura; git --git-dir=.cfg ls-files | while IFS= read -r f; do cmp -s \"$S/\$f\" \"\$f\" || echo \"DISTINTO: \$f\"; done"
```

| Comprobación | Resultado |
|---|---|
| Contenido de los 397 archivos trackeados | ✅ idénticos al snapshot, salvo `README.md` (esperado: créditos añadidos después del snapshot) |
| `niri validate` como `sura` | ✅ `config is valid` |
| Thunderbird (`--headless` como `sura`) | ✅ usa el perfil `ekf5x1zc…` del repo, no crea otro, `prefs.js` con la pref de stylesheets |
| GTK3/GTK4 `settings.ini` | ✅ MacTahoe-Dark, iconos, cursor, botones a la izquierda |
| Clavis arranca | ✅ vía `spawn-at-startup "key" "shell" "--daemon"` de `config.kdl` (sin los servicios systemd que tiene `niri`) |
| Plantillas Matugen de Clavis | ❌ `Template file is missing or not a readable regular file` para `fcitx5_*`, `clavis_fish`, `niri_focus_ring`: el registro apunta a `/home/niri/...` |
| `~/.config/gtk-4.0/gtk.css` y `gtk-dark.css` | ❌ symlinks absolutos a `/home/niri/.config/gtk-4.0/gtk-Dark.css` (libadwaita sin tema) |
| `config.json` de Clavis | ❌ `wallpaper.folder` y `wallpaper.path` en `/home/niri/...` |
| `obsidian.json` | ❌ vault registrado en `/home/niri/Documents/Notas` |
| gsettings (`dconf`) | ❌ `sura`: Adwaita / `default` / `appmenu:close`; `niri`: MacTahoe-Dark / `prefer-dark` / `close,minimize,maximize:` |

Lo que tiene `niri` y **no** está en el repo (fuera de backups, historial, cachés y salidas generadas):

| Falta | Efecto |
|---|---|
| `~/.config/dconf/user` (gsettings) | apps GTK/libadwaita y portal no usan MacTahoe ni modo oscuro |
| `~/.config/systemd/user/niri.service.wants/clavis-{shell,clipboard}.service` | Clavis sin supervisión/reinicio de systemd; clipboard sin `key clipboard watch` |
| `~/.icons/default/index.theme` (`Inherits=MacTahoe-dark-cursors`) | cursor por defecto en apps XWayland |
| `~/Pictures/mpv` | carpeta de capturas de mpv |

Prueba de corrección (en `sura`, luego revertida): con `sed -i "s|/home/niri|~|g" ~/.config/clavis/matugen/config.toml` Clavis genera sin errores `~/.config/clavis-fish-theme/colors.conf` y `~/.local/share/fcitx5/themes/Matugen/`. **Clavis acepta `~` en `input_path`.**

## 4️⃣ Conclusiones

1. La instalación con dotbare funciona: todo llega íntegro y niri valida.
2. El repo **no es portable a otro usuario**: hay rutas absolutas `/home/niri` (11 en 3 archivos + 2 symlinks). Con `niri` en otra máquina funcionaría; con cualquier otro nombre de usuario no.
3. Parte del estado vive **fuera de archivos versionables** (gsettings/dconf, servicios systemd habilitados, carpetas vacías). Hace falta un paso post-instalación.

## 5️⃣ Correcciones: [PR #1](https://github.com/robert-flo/niri-dotfiles/pull/1) "Make install portable to any user" (mergeado, `2c454b0`)

| Problema | Corrección |
|---|---|
| Registro Matugen con `/home/niri/...` | `~/...` (Clavis expande `~` en `input_path`; probado) |
| `config.json` de Clavis: `wallpaper.path` absoluto | Clavis **no** expande `~` ahí (`PersonalizationConfig.qml` usa la cadena tal cual) → lo adapta el script post-install |
| `obsidian.json` con vault en `/home/niri` | lo adapta el script post-install |
| symlinks absolutos de `gtk-4.0` | relativos: `gtk.css -> gtk-Dark.css`, `gtk-dark.css -> gtk-Dark.css` |
| cursor por defecto | se trackea `~/.icons/default/index.theme` (`Inherits=MacTahoe-dark-cursors`) |
| gsettings, servicios, carpetas, colores | nuevo `~/.local/bin/niri-dotfiles-postinstall` (idempotente) |

**¿Por qué el cursor por defecto?** `cursor-theme.kdl` le dice a niri qué cursor dibujar y exporta `XCURSOR_THEME`/`XCURSOR_SIZE` a las apps que **niri lanza**. Las apps que no heredan ese entorno (activadas por D-Bus o systemd, algunas X11 vía xwayland-satellite, o lanzadas con entorno limpio) buscan el tema `default` en `~/.icons/default/index.theme` y, si no existe, en `/usr/share/icons/default/index.theme`, que en este sistema dice `Inherits=Adwaita`. Sin el archivo, esas apps muestran el cursor Adwaita encima de sus ventanas. Es el mismo archivo que se creó para `niri` en la sección 3 del playbook principal.

`niri-dotfiles-postinstall`:

1. Reemplaza `/home/niri` por `$HOME` en `~/.config/clavis/config.json` y `~/.config/obsidian/obsidian.json` (solo si el usuario no es `niri`).
2. `gsettings`: `gtk-theme MacTahoe-Dark`, `icon-theme MacTahoe-dark`, `cursor-theme MacTahoe-dark-cursors`, `cursor-size 24`, `color-scheme prefer-dark`, `button-layout close,minimize,maximize:`. Si no hay bus D-Bus, avisa.
3. `systemctl --user enable clavis-shell.service clavis-clipboard.service`.
4. `mkdir -p ~/Pictures/mpv`.
5. Genera los colores Matugen con las plantillas activas en `theme.matugenTemplates` y el wallpaper de `config.json`.
6. `niri validate`.

Consecuencia esperada: en un usuario distinto de `niri`, `dotbare status` muestra `M` en `config.json` y `obsidian.json` (rutas locales). No commitearlas.

Antes de mergear, el script se probó como `niri`: sin cambios inesperados, colores generados con rutas `~`, config válida.

## 6️⃣ Segunda prueba (rama del PR sobre `sura` limpio)

Reinicio de `sura` (equivale a un usuario recién creado):

```bash
sudo systemctl stop user@1002.service                         # linger lo mantiene vivo
sudo find /home/sura -mindepth 1 -maxdepth 1 -exec rm -rf {} +
sudo cp -a /etc/skel/. /home/sura/ && sudo chown -R sura:sura /home/sura
sudo systemctl start user@1002.service
```

Instalación y post-install:

```bash
sudo -u sura env HOME=/home/sura DOTBARE_DIR=/home/sura/.cfg DOTBARE_TREE=/home/sura bash -c 'cd ~
dotbare finit -y -u https://github.com/robert-flo/niri-dotfiles.git </dev/null
git --git-dir=.cfg --work-tree=. checkout portable-install'       # solo para probar el PR antes de mergear
sudo -u sura env HOME=/home/sura XDG_RUNTIME_DIR=/run/user/1002 ~/.local/bin/niri-dotfiles-postinstall
```

| Comprobación | Resultado |
|---|---|
| Archivos trackeados vs snapshot | ✅ iguales salvo los cambios intencionados (README, registro con `~`, rutas adaptadas, script nuevo) |
| `/home/niri` en archivos trackeados | ✅ solo dentro del propio script (patrón del `sed`) |
| gsettings | ✅ MacTahoe-Dark / MacTahoe-dark / MacTahoe-dark-cursors / `prefer-dark` / `close,minimize,maximize:` |
| Servicios Clavis | ✅ `enabled` |
| `gtk.css` (libadwaita) | ✅ relativo y legible |
| Wallpaper y vault de Obsidian | ✅ `/home/sura/...`, el wallpaper existe |
| Colores Matugen (fish, fcitx5 `theme.conf`/`panel.svg`, focus ring) | ✅ **idénticos byte a byte** a los de `niri` |
| `niri validate` | ✅ |
| Symlinks rotos | solo `~/.config/nvim/lua/plugins/theme.lua` del skel de Omarchy (ajeno al repo) |

Después: PR #1 mergeado con `--rebase` (rama 0 commits detrás de `master`), rama `portable-install` borrada en GitHub y en los repos bare de `niri` y `sura`; ambos en `master` `2c454b0`.

## 7️⃣ Conclusiones finales

- La instalación es portable: un usuario nuevo queda igual que `niri` con `dotbare finit -u` + `niri-dotfiles-postinstall`.
- Pendiente de comprobar **visualmente** iniciando sesión en niri como `sura` (lo que no se puede verificar sin sesión: Clavis en pantalla, blur, cursor, apps).
- `sura` se conserva como **banco de pruebas** para no tocar el usuario real: reiniciarlo con los comandos de la sección 6 antes de cada prueba.

## 7️⃣.1 ⏳ Pendiente: revisión visual en sesión `sura` (Niri)

Iniciar sesión como `sura` eligiendo la sesión **Niri** y marcar:

- [ ] **Clavis:** aparecen la barra y los paneles, y el wallpaper es `wallhaven-5y3571`.
- [ ] **Ventanas:** el focus ring tiene el degradado cian → lavanda, y abrir y minimizar ventanas tiene animación.
- [ ] **Cursor:** MacTahoe, también encima de apps como Thunderbird u Obsidian.
- [ ] **Apps GTK** (p. ej. Nautilus): tema MacTahoe-Dark, con los botones a la izquierda.
- [ ] **Terminal:** Alacritty abre fish con el prompt de colores.
- [ ] **Apps:**
  - [ ] mpv con la interfaz uosc;
  - [ ] Thunderbird con el tema LiquidBird;
  - [ ] Obsidian con el tema Violet y el vault "Notas" (vacío: las notas no están en el repo).
- [ ] **Método de entrada:** el panel de fcitx5 usa los colores del wallpaper.
- [ ] **Cambiar el wallpaper desde Clavis:** los colores del focus ring, fish y fcitx5 se actualizan solos.

En el primer inicio de sesión el keyring puede pedir crear una contraseña; es normal.

Después de la sesión: revisar qué cambió con `dotbare diff` (como `sura`) y comparando su home con el snapshot, y documentar aquí los resultados.

## 8️⃣ Eliminar `sura` (si algún día ya no hace falta)

```bash
sudo loginctl disable-linger sura
sudo userdel -r sura
sudo rm -f /etc/sudoers.d/04_sura
```
