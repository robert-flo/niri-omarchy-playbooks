# Playbook: dotfiles de `niri` con dotbare → GitHub `robert-flo/niri-dotfiles`

Repo público: https://github.com/robert-flo/niri-dotfiles (rama `master`, commit inicial `04eb5ce`, reescrito con atribución; antes `9990c43`).
Estado = snapshot Btrfs `home-2026-10-08-dotfiles` (ver `playbook-snapshot-home-btrfs.md`), antes del primer inicio de sesión de `niri`.

Decisiones del usuario: [dotbare](https://github.com/kazhala/dotbare) (repo *bare* sobre `$HOME`), repo **público**, incluir `~/Q/m2-res_1080p.mp4` como showcase, atribución al autor original desde el commit 1 (sección 8), y solo **dos wallpapers** (`wallhaven-5y3571.jpg`, `wallhaven-rq6yqm.jpg`).

---

## 1️⃣ Preparación

```bash
yay -S --needed dotbare          # AUR, solo depende de fzf
```

Correcciones previas en `/home/niri` (necesarias para que el repo sea correcto):

- `~/.local/share/fcitx5-matugen-theme/.git` **eliminado**: git ignora en silencio archivos dentro de un repo anidado, y dotbare no podía trackearlos. (Copia original en el snapshot `home-2026-10-08-statindet`.)
- Thunderbird: el tema estaba en un perfil que no se usaba. Se movió a `ekf5x1zc.default-release-1` (ver corrección en la sección 8.3 de `playbook-mactahoe-iconos-cursor-wallpaper.md`).
- `~/README.md` y `~/.github/showcase.mp4` (copia de `~/Q/m2-res_1080p.mp4`), dueño `niri`.

## 2️⃣ Crear el repo bare (como `niri`)

```bash
sudo -u niri env HOME=/home/niri DOTBARE_DIR=/home/niri/.cfg DOTBARE_TREE=/home/niri \
  GIT_CONFIG_PARAMETERS="'init.defaultBranch=master'" bash -c 'cd ~
dotbare finit -y </dev/null            # git init --bare ~/.cfg + status.showUntrackedFiles=no
git --git-dir=.cfg config user.name  "Roberto Flores"
git --git-dir=.cfg config user.email "25asab015@ujmd.edu.sv"'
```

(Sin TTY, dotbare imprime `/dev/tty: No such device or address`; es inofensivo.)

## 3️⃣ Qué se trackea (lista curada, 397 archivos)

```bash
sudo -u niri env HOME=/home/niri DOTBARE_DIR=/home/niri/.cfg DOTBARE_TREE=/home/niri bash -c 'cd ~
dotbare add README.md .github/showcase.mp4 \
  .config/niri/config.kdl .config/niri/clavis .config/niri/cursor-theme.kdl .config/niri/obsidian-violet.kdl \
  .config/niri/liquidbird.kdl .config/niri/statindet-animations.kdl .config/niri/statindet-colors.kdl \
  .config/clavis/config.json .config/clavis/matugen/config.toml \
  .local/share/statindet-niri .local/share/fcitx5-matugen-theme .local/share/clavis-fish-theme \
  .config/mpv .config/gtk-3.0/settings.ini .config/gtk-4.0 .config/environment.d \
  .config/fcitx5/conf/classicui.conf .config/fish/config.fish .config/alacritty/alacritty.toml \
  .config/thunderbird/profiles.ini .config/thunderbird/installs.ini \
  .config/thunderbird/ekf5x1zc.default-release-1/chrome .config/thunderbird/ekf5x1zc.default-release-1/user.js \
  .config/obsidian/obsidian.json .config/obsidian/user-flags.conf \
  Documents/Notas/.obsidian/appearance.json Documents/Notas/.obsidian/themes Documents/Notas/.obsidian/snippets \
  .local/share/clavis/wallpapers/wallhaven-5y3571.jpg .local/share/clavis/wallpapers/statindet/wallhaven-rq6yqm.jpg'
```

**No** se incluye: backups `*.bak*` / `*.clavis-backup-*`, datos del perfil de Thunderbird (correo, contraseñas), notas de Obsidian, cachés, `.local/state`, salidas generadas por Matugen salvo `statindet-colors.kdl` (necesario para que `config.kdl` valide en una instalación nueva), y el resto de wallpapers.

## 4️⃣ Revisión de secretos (repo público)

```bash
sudo -u niri bash -c 'cd ~; git --git-dir=.cfg --work-tree=. ls-files -z | xargs -0 rg -n -i -I \
  -e "ghp_[A-Za-z0-9]{20,}" -e "github_pat_" -e "-----BEGIN .*PRIVATE KEY" \
  -e "(password|passwd|secret|api[_-]?key|token)\s*[=:]\s*\S{6,}" -e "@(gmail|outlook|hotmail)\."'
```

Único hallazgo: `open_subtitles_api_key` en `.config/mpv/scripts/uosc/main.lua`, clave pública que trae uosc 5.13.0 en su código oficial. Aceptado.

## 5️⃣ Commit y push

```bash
sudo -u niri bash -c 'cd ~; git --git-dir=.cfg --work-tree=. commit -m "Initial commit: niri + Clavis desktop configuration"'
gh repo create niri-dotfiles --public --description "niri + Clavis (Quickshell) desktop dotfiles, managed with dotbare"

# push como niri usando el token de gh de gracie por stdin (no queda guardado ni aparece en ps)
gh auth token | sudo -u niri bash -c 'read -r T; export T; cd ~; G="git --git-dir=.cfg --work-tree=."
$G remote add origin https://github.com/robert-flo/niri-dotfiles.git
$G -c credential.helper= -c "credential.helper=!f(){ echo username=x-access-token; echo password=\$T; }; f" push -u origin master'
```

Reglas del repo: el commit inicial va directo a `master`; **todo cambio posterior** en rama auxiliar → PR a `master` → rebase sobre `master` antes de mergear → borrar la rama local y remota. Sin `Co-authored-by`.

## 6️⃣ Uso diario (como `niri`, dentro de su sesión)

```bash
export DOTBARE_DIR="$HOME/.cfg" DOTBARE_TREE="$HOME"   # p. ej. en ~/.config/fish/config.fish con set -gx
dotbare fstat          # ver cambios con fzf
dotbare fadd           # elegir archivos a añadir
dotbare checkout -b mi-cambio && dotbare commit -m "..." && dotbare push -u origin mi-cambio
gh pr create --repo robert-flo/niri-dotfiles --base master --head mi-cambio
```

Tras el primer login: `dotbare diff` muestra qué cambió la sesión en los archivos trackeados; decidir si se commitea (en rama + PR) o se restaura con `dotbare checkout -- <archivo>`.

## 7️⃣ Instalar en otra máquina

```bash
yay -S dotbare
export DOTBARE_DIR="$HOME/.cfg" DOTBARE_TREE="$HOME"
dotbare finit -u https://github.com/robert-flo/niri-dotfiles.git
~/.local/bin/niri-dotfiles-postinstall      # rutas a $HOME, gsettings, servicios Clavis, colores Matugen, validación
```

Probado en un usuario limpio (`sura`): ver `playbook-prueba-instalacion-sura.md`.

Requisitos globales previos: niri-edge + Clavis (`playbook-paso-1..3`), temas MacTahoe (secciones 1–5 del playbook principal), paquetes `mpv thunderbird obsidian fish alacritty fcitx5 matugen ttf-lxgw-wenkai-screen ttf-jetbrains-mono-nerd`.

## 8️⃣ Atribución desde el commit 1 (reescritura de historial)

Decisión del usuario: no es un fork (combina muchos repos), pero la atribución debe existir desde el primer commit. El README incluye:

- Aviso al principio: basado en el escritorio de [StatIndet](https://github.com/StatIndet) mostrado en [r/unixporn](https://www.reddit.com/r/unixporn/s/4QCJSEDKQ7).
- Sección **Créditos y atribución**: tabla de los repos del autor (quickshell/Clavis, key-cli, niri-edge, dotfiles, mpv, obsidian-violet, liquidbird, fcitx5-matugen-theme, clavis-fish-theme, wallpaper) y de otros proyectos (niri, Quickshell, Matugen, uosc, MacTahoe, dotbare, Wallhaven 5y3571/rq6yqm).
- Nota de licencias, verificadas con `gh api repos/<repo> -q .license.spdx_id`: uosc LGPL-2.1, MacTahoe icon GPL-3.0, MacTahoe GTK **MIT**, fcitx5-matugen-theme / quickshell / key-cli / niri GPL-3.0, Matugen GPL-2.0.

Reescritura (había un solo commit):

```bash
sudo install -o niri -g niri -m 644 /tmp/niri-README.md /home/niri/README.md
sudo -u niri bash -c 'cd ~; G="git --git-dir=.cfg --work-tree=."; $G add README.md
$G commit --amend -m "Initial commit: niri + Clavis desktop configuration

Based on the niri desktop by StatIndet shown on r/unixporn:
https://www.reddit.com/r/unixporn/s/4QCJSEDKQ7
See README.md for full credits and attribution."'
# push forzado solo si el remoto sigue en el commit esperado
gh auth token | sudo -u niri bash -c 'read -r T; export T; cd ~; G="git --git-dir=.cfg --work-tree=."
$G -c credential.helper= -c "credential.helper=!f(){ echo username=x-access-token; echo password=\$T; }; f" \
  push --force-with-lease=master:9990c43 origin master'
```

Resultado: `9990c43...04eb5ce master -> master (forced update)`; GitHub muestra 1 commit (`04eb5ce`).

Los nombres `statindet-*.kdl` y las licencias dentro de uosc / LiquidBird / fcitx5-matugen-theme se conservan tal cual.

## 9️⃣ PR #1: instalación portable (2026-10-08)

[PR #1](https://github.com/robert-flo/niri-dotfiles/pull/1), mergeado con `--rebase` → `master` en `2c454b0`. Registro Matugen con `~`, symlinks GTK4 relativos, `.icons/default/index.theme`, script `~/.local/bin/niri-dotfiles-postinstall` y README actualizado. Detalle y pruebas en `playbook-prueba-instalacion-sura.md`.

Flujo usado (reglas del repo): rama `portable-install` en el repo bare de `niri` → push → `gh pr create` → prueba en `sura` → comprobar `behind_by=0` contra `master` → `gh pr merge --rebase --delete-branch` → `git fetch --prune`, `checkout master`, `merge --ff-only origin/master`, `branch -D portable-install` en `niri` y `sura`.
