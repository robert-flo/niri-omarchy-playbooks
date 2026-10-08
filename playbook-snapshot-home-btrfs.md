# Playbook: Snapshot de `/home` con Btrfs (interno + disco externo)

Guarda el estado de `/home` (incluye `/home/niri` con todo lo de `playbook-mactahoe-iconos-cursor-wallpaper.md`) en un snapshot de solo lectura en el disco interno y una copia exacta en el disco externo con `btrfs send | btrfs receive`.

---

## 🧭 Contexto

| Elemento | Valor |
|---|---|
| Disco interno | `/dev/mapper/root` (LUKS + Btrfs), subvolúmenes `@`, `@home`, `@log`, `@pkg` |
| `/home` | subvolumen `@home` |
| `/home/niri` | **carpeta normal dentro de `@home`** (no es subvolumen), por eso se hace snapshot de todo `@home` |
| Disco externo | `ALMACEN_BTRFS` (USB, LUKS + Btrfs), montado en `/run/media/gracie/ALMACEN_BTRFS` |
| snapper | solo tiene config `root` (`/`); no gestiona `/home` |
| Herramientas | `btrfs-progs` (ya instalado) |

Tamaños el 2026-10-08: `gracie` 14 GB, `niri` 226 MB.

---

## 1️⃣ Snapshot interno (solo lectura)

```bash
N=home-$(date +%F)-statindet
sudo mkdir -p /home/.snapshots && sudo chmod 700 /home/.snapshots
sudo btrfs subvolume snapshot -r /home /home/.snapshots/$N
sudo btrfs subvolume show /home/.snapshots/$N | grep -Ei "name|flags|creation"   # Flags: readonly
```

- Es instantáneo y al principio no ocupa espacio (comparte bloques con `/home`; crece según cambien los archivos).
- `-r` (solo lectura) es obligatorio para poder usar `btrfs send`.
- Es consistente "como un corte de luz": si hay apps abiertas escribiendo (navegador, etc.) se guarda el estado de ese instante.

Snapshot hecho: `/home/.snapshots/home-2026-10-08-statindet` (15:21, `Flags: readonly`).

## 2️⃣ Copia al disco externo (`send | receive`)

```bash
N=home-2026-10-08-statindet
E=/run/media/gracie/ALMACEN_BTRFS/gracie/snapshots/home
sudo mkdir -p $E
sudo btrfs send /home/.snapshots/$N | sudo btrfs receive $E      # ~2 min 40 s para 14 GB por USB
sudo btrfs subvolume show $E/$N | grep -Ei "name|received uuid|flags"
```

Verificación (copia idéntica, dueños conservados):

```bash
sudo diff -rq --no-dereference /home/.snapshots/$N/niri $E/$N/niri && echo niri_identico
sudo stat -c '%U:%G %n' $E/$N/niri/.config/niri/config.kdl     # niri:niri
```

Resultado: `niri_identico`, dueño `niri:niri`, `Received UUID` presente y `Flags: readonly`.

Segundo snapshot `home-2026-10-08-dotfiles` (tras corregir el perfil de Thunderbird y crear el repo de dotfiles), enviado **incremental** con `-p home-2026-10-08-statindet` en 7 s. Es el que corresponde al commit `9990c43` (antes de añadir créditos al README; el commit actual es `04eb5ce`) de [robert-flo/niri-dotfiles](https://github.com/robert-flo/niri-dotfiles). Para comparar tras el primer login usar este (`S=/home/.snapshots/home-2026-10-08-dotfiles/niri`).

### 2.1 Copia "navegable" en el disco externo (ACL para cualquier usuario)

Los snapshots recibidos son de solo lectura (`ro=true`, necesario para los envíos incrementales) y dentro conservan permisos `700` por usuario, así que `gracie` no puede abrir `niri/` ni escribir. La carpeta `snapshots/` ya tenía ACL `rwx` + default para cualquier usuario; eso no alcanza el interior de un subvolumen ro. Solución: un snapshot **escribible** del recibido (instantáneo, CoW, sin espacio extra) y ACL sobre esa copia:

```bash
E=/run/media/gracie/ALMACEN_BTRFS/gracie/snapshots; N=home-2026-10-08-dotfiles; B=$E/navegable
sudo mkdir -p $B
sudo btrfs subvolume snapshot $E/home/$N $B/$N                       # sin -r → ro=false
sudo setfacl -R -P -m u::rwX,g::rwX,o::rwX,m::rwX $B/$N               # cualquier usuario: leer/escribir/borrar
sudo find $B/$N -type d -exec setfacl -d -m u::rwx,g::rwx,o::rwx,m::rwx {} +   # heredado por lo nuevo
```

- Verificado: `gracie` lista `niri/.config/niri` y crea/borra archivos; el snapshot original sigue `ro=true`.
- Cambios en la copia navegable no afectan a los snapshots ni a la cadena incremental. Lo que se escriba en ella sí ocupa espacio nuevo.
- Borrar la copia completa: `sudo btrfs subvolume delete $B/$N` (un usuario normal puede vaciarla, pero no eliminar el subvolumen en sí).
- Para un snapshot más nuevo, repetir con su nombre.

## 3️⃣ Próximas copias (incrementales)

Mantener **el último snapshot en ambos discos** como "padre"; `send -p` solo transfiere los cambios:

```bash
P=home-2026-10-08-statindet                 # padre: debe existir en los dos discos
N=home-$(date +%F)-algo
E=/run/media/gracie/ALMACEN_BTRFS/gracie/snapshots/home
sudo btrfs subvolume snapshot -r /home /home/.snapshots/$N
sudo btrfs send -p /home/.snapshots/$P /home/.snapshots/$N | sudo btrfs receive $E
```

## 4️⃣ Restaurar solo `/home/niri`

Con el usuario `niri` sin sesión abierta (`loginctl list-sessions`; si quedan procesos: `sudo systemctl stop user@1001`):

```bash
S=/home/.snapshots/home-2026-10-08-statindet      # o la ruta del disco externo
sudo mv /home/niri /home/niri.antes-de-restaurar
sudo cp -a --reflink=auto $S/niri /home/niri      # reflink: instantáneo si es el disco interno
# revisar y luego: sudo rm -rf /home/niri.antes-de-restaurar
```

Un solo archivo o carpeta: `sudo cp -a $S/niri/.config/niri /home/niri/.config/`.

## 4️⃣.5 Primer inicio de sesión de `niri`: qué puede cambiar y cómo comprobarlo

Revisado antes del primer login (2026-10-08):

| Qué arranca | ¿Sobrescribe lo configurado? |
|---|---|
| `omarchy-provision-first-run` | **No se ejecuta**: solo lo lanza el autostart de Hyprland (`/usr/share/omarchy/default/hypr/autostart.lua`); la sesión `niri` usa `niri-session` |
| `.bashrc` / `.bash_profile` (Omarchy) | No: solo aliases, funciones y PATH |
| Clavis (`key shell --daemon`) | No hay asistente de primer uso. `config.json` solo se reescribe al cambiar ajustes y **conserva** `theme.matugenTemplates`. `niri_config.py` solo actúa al configurar algo desde Clavis, añade sus `include "clavis/..."` y no toca nuestros includes |
| Matugen de Clavis al arrancar | Regenera **salidas generadas** (esperado): `statindet-colors.kdl`, tema fcitx5, colores de fish. Sus plantillas internas (btop, cava, kitty, yazi) escriben archivos que hoy no existen: los crea, no pisa nada |
| fcitx5, xdg-user-dirs, gnome-keyring | fcitx5 puede reescribir sus `.conf` conservando valores; xdg-user-dirs solo crea carpetas; keyring pide crear contraseña |
| Obsidian / Thunderbird | Obsidian añade datos a `obsidian.json`; Thunderbird reaplica `user.js` en cada arranque |

Después del primer login, comparar con el snapshot para ver exactamente qué cambió:

```bash
S=/home/.snapshots/home-2026-10-08-dotfiles/niri
sudo diff -rq --no-dereference $S /home/niri 2>/dev/null \
  | grep -vE '/\.cache/|/\.local/state/|/\.local/share/(recently-used|Trash)|\.mozilla|/\.config/(Electron|obsidian/(Cache|GPUCache|Code Cache))' | head -50
# Restaurar un archivo concreto:  sudo cp -a $S/<ruta> /home/niri/<ruta>
```

## 5️⃣ Listar y borrar snapshots

```bash
sudo ls /home/.snapshots /run/media/gracie/ALMACEN_BTRFS/gracie/snapshots/home
sudo btrfs subvolume delete /home/.snapshots/<nombre>
sudo btrfs subvolume delete /run/media/gracie/ALMACEN_BTRFS/gracie/snapshots/home/<nombre>
```

No borres el último snapshot común a ambos discos si quieres seguir haciendo copias incrementales.

## ⚠️ Notas

- Los snapshots internos **no son backup** si falla el disco interno; para eso está la copia externa.
- Los snapshots retienen espacio: si borras archivos grandes de `/home`, el espacio no se libera hasta borrar los snapshots que los contienen.
- Alternativa no usada: convertir `/home/niri` en subvolumen propio (snapshots de solo ese usuario y config de snapper `niri`). Se eligió snapshot de todo `@home` para no tocar la estructura.
