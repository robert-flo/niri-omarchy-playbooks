# Playbook: Corregir parpadeo de pantalla en Intel i915 (FIFO Underrun / SAGV)

Guía paso a paso y playbook automatizado para ejecutar a mano o pasárselo a un agente de IA (Cursor, Claude Code, Aider, etc.) en un equipo con **Omarchy / Arch Linux + Hyprland** y gráfica integrada Intel.

---

## 🎯 Objetivo

Eliminar los parpadeos negros breves de la pantalla causados por *FIFO underruns* del controlador `i915`, desactivando SAGV y fijando el monitor de forma explícita en Hyprland.

---

## 🩺 Diagnóstico

Síntoma: la pantalla se pone negra durante una fracción de segundo, sobre todo al mover el cursor o cuando el equipo está en reposo.

Confirmación en el log del kernel:

```bash
journalctl -k | grep -i "fifo underrun"
# i915 0000:00:02.0: [drm] *ERROR* CPU pipe A FIFO underrun
```

**Causa raíz:** en Intel Skylake (por ejemplo, HD Graphics 530), **SAGV** (System Agent Geyserville) cambia dinámicamente el voltaje y la frecuencia de la RAM. Cada cambio introduce un pico de latencia que vacía el búfer de pantalla (FIFO underrun) y se pierde la señal un instante.

Comprobar el estado actual:

```bash
sudo cat /sys/module/i915/parameters/enable_sagv   # Y = activo (problema)
modinfo -p i915 | grep sagv                         # confirma que el parámetro existe
hyprctl monitors all                                # salida, modo y tasa soportada
```

> No es un problema de tasa de refresco: el Dell P2417H solo soporta 60 Hz a 1080p según su EDID.

---

## ⚠️ Caso límite: initramfs / UKI

En Omarchy, `i915` se carga **dentro del initramfs** (hook `kms` en `/etc/mkinitcpio.conf`). Con solo crear `/etc/modprobe.d/i915.conf` **no basta**: hay que regenerar el initramfs con `mkinitcpio -P`. Omarchy usa Limine con una imagen de kernel unificada (UKI) en `/boot/EFI/Linux/omarchy_linux-omarchy.efi`, que también se regenera en ese paso.

---

## 🤖 Prompt para el agente

```text
Corrige el parpadeo de pantalla por "FIFO underrun" del driver Intel i915 en mi Omarchy/Hyprland:
1. Confirma el error con: journalctl -k | grep -i "fifo underrun"
2. Confirma que existe el parámetro: modinfo -p i915 | grep sagv
3. Crea /etc/modprobe.d/i915.conf con: options i915 enable_sagv=0
4. Regenera el initramfs/UKI con: sudo mkinitcpio -P (i915 se carga en early KMS)
5. Verifica que i915.conf quedó dentro del initrd de la UKI.
6. En ~/.config/hypr/monitors.lua (haz backup antes), agrega una regla fija para el monitor
   detectado con hyprctl monitors, por ejemplo:
   hl.monitor({ output = "DP-1", mode = "1920x1080@60", position = "0x0", scale = 1 })
7. Valida con hyprctl reload && hyprctl configerrors.
8. Pídeme reiniciar y luego verifica que enable_sagv = N.
```

---

## 🛠️ Ejecución automatizada (script Bash idempotente)

Ajusta `OUTPUT` y `MODE` a tu monitor (`hyprctl monitors`).

```bash
set -euo pipefail
OUTPUT="DP-1"
MODE="1920x1080@60"
MON="$HOME/.config/hypr/monitors.lua"

# A. Desactivar SAGV en i915
echo "options i915 enable_sagv=0" | sudo tee /etc/modprobe.d/i915.conf
sudo mkinitcpio -P

# B. Fijar el monitor en Hyprland
cp "$MON" "$MON.bak.$(date +%s)"
grep -q "output = \"$OUTPUT\"" "$MON" || \
  echo "hl.monitor({ output = \"$OUTPUT\", mode = \"$MODE\", position = \"0x0\", scale = 1 })" >> "$MON"
hyprctl reload && hyprctl configerrors

echo "✅ Listo. Reinicia el equipo para aplicar enable_sagv=0."
```

---

## 🔍 Verificación

1. **La regla está dentro del initrd de la UKI (antes de reiniciar):**
   ```bash
   sudo objcopy -O binary --only-section=.initrd /boot/EFI/Linux/omarchy_linux-omarchy.efi /tmp/initrd.img
   sudo lsinitcpio /tmp/initrd.img | grep i915.conf && sudo rm /tmp/initrd.img
   ```
2. **Tras reiniciar, SAGV está desactivado:**
   ```bash
   sudo cat /sys/module/i915/parameters/enable_sagv   # debe mostrar N
   ```
3. **No aparecen nuevos underruns en el arranque actual:**
   ```bash
   journalctl -k -b | grep -i "fifo underrun"   # sin resultados
   ```
4. **Monitor fijo:** `hyprctl monitors` muestra `1920x1080@60.00000 at 0x0`, `scale: 1`.

---

## ➕ Si el parpadeo persiste

Desactivar el cursor por hardware en Hyprland (en `~/.config/hypr/looknfeel.lua` o equivalente). Valores: `0` = usar cursor por hardware, `1` = desactivarlo, `2` = automático (por defecto):

```lua
hl.config({ cursor = { no_hardware_cursors = 1 } })
```

---

## 🪟 Sesión niri (usuario `niri`)

La corrección de SAGV (`/etc/modprobe.d/i915.conf` + `mkinitcpio -P`) es **a nivel de kernel**, así que también cubre la sesión niri sin hacer nada más. Lo único específico de niri es fijar el monitor y, si hace falta, desactivar el plano de cursor por hardware.

**Contexto de este equipo:**
- La configuración vive en `/home/niri/.config/niri/config.kdl` (propiedad `niri:niri`).
- El bloque `output` de fábrica está comentado (`/-output "eDP-1" { ... }`), así que niri detecta el monitor automáticamente.
- `clavis/*.kdl` está **gestionado por Clavis** (se sobrescribe). No pongas ahí la configuración del monitor: usa un archivo propio, `outputs.kdl`, cargado con `include`.

### 1. Identificar la salida (dentro de la sesión niri)

```bash
niri msg outputs   # nombre (DP-1), modos disponibles, escala actual
```

### 2. Fijar el monitor

```bash
sudo -u niri -H bash -c '
set -euo pipefail
DIR="$HOME/.config/niri"
cp "$DIR/config.kdl" "$DIR/config.kdl.bak.$(date +%s)"

cat > "$DIR/outputs.kdl" <<EOF
output "DP-1" {
    mode "1920x1080@60.000"
    scale 1
    transform "normal"
    position x=0 y=0
}
EOF

grep -q "include \"outputs.kdl\"" "$DIR/config.kdl" || \
  printf "\ninclude \"outputs.kdl\";\n" >> "$DIR/config.kdl"

niri validate -c "$DIR/config.kdl"
'
```

> Ajusta `mode` al valor exacto que muestra `niri msg outputs` (por ejemplo, `1920x1080@60.000`). Si el modo no coincide, niri elige uno automáticamente. niri recarga la configuración en caliente al guardar.

### 3. Si el parpadeo persiste: desactivar el plano de cursor por hardware

Es el equivalente en niri a `no_hardware_cursors` de Hyprland. Agrégalo a `outputs.kdl` (o a `config.kdl`, fuera de `clavis/`):

```kdl
debug {
    disable-cursor-plane
}
```

### Verificación en niri

```bash
niri msg outputs                              # DP-1 a 1920x1080@60, escala 1
sudo cat /sys/module/i915/parameters/enable_sagv  # N tras reiniciar
journalctl -k -b | grep -i "fifo underrun"    # sin resultados
```

---

## ↩️ Revertir

```bash
sudo rm /etc/modprobe.d/i915.conf && sudo mkinitcpio -P
# Restaurar monitors.lua desde el backup más reciente:
ls -t ~/.config/hypr/monitors.lua.bak.* | head -1 | xargs -I{} cp {} ~/.config/hypr/monitors.lua
# niri: quitar el include y el archivo de salidas
sudo -u niri -H bash -c 'sed -i "/include \"outputs.kdl\";/d" ~/.config/niri/config.kdl && rm -f ~/.config/niri/outputs.kdl'
```
