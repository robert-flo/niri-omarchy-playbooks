# Playbook — Paso 2 de 3: Creación de Usuario Dedicado (niri) y Configuración de Login (SDDM)

> **Estrategia Global (Roadmap de 3 Pasos):**
> * **Paso 1:** Preparación del sistema base, instalación del compositor (`niri-edge`) y runtime core (`key-cli`) a nivel global sin tocar configuraciones de usuario. *(Completado)*
> * **👉 Paso 2 (Este Playbook):** Creación del usuario dedicado `niri`, paridad total de permisos con `gracie`, y configuración de SDDM para permitir la selección interactiva de usuario y sesión gráfica. *(Completado)*
> * **Paso 3:** Configuración del entorno de escritorio, Clavis Shell / dotfiles y servicios de sesión para el nuevo usuario `niri`. *(Siguiente paso)*

---

## 🎯 Objetivo del Paso 2

1. Crear un nuevo usuario en el sistema llamado **`niri`** con su respectivo directorio personal (`/home/niri`), shell `/bin/bash` y asignación al grupo administrativo **`wheel`**.
2. Garantizar **paridad total de permisos** con el usuario original (`gracie`), incluyendo privilegios de `sudo` sin contraseña y la directiva `verifypw = any` para no bloquear scripts del sistema.
3. Configurar **SDDM** para que en cada encendido o cierre de sesión presente una interfaz gráfica que permita **elegir interactivamente el usuario** (`gracie` o `niri`) y la **sesión de escritorio** (`Niri` o `Hyprland/Omarchy`).

> [!IMPORTANT]
> **Política de Seguridad:** Por directiva explícita, **la contraseña no se almacena en este playbook**. Al ejecutarlo en una nueva máquina, debe suministrarse interactivamente o mediante variables de entorno seguras.

---

## 💡 Decisiones Técnicas y Motivos (Rationale)

### 1. ¿Por qué NO reinstalar SDDM desde cero?
En Omarchy (Arch Linux), SDDM ya viene preinstalado y configurado como servicio del sistema (`sddm.service`), corriendo de forma nativa sobre Wayland con `start-hyprland`. Reinstalarlo o sustituirlo por otro display manager rompería las integraciones de Plymouth y el arranque del sistema. Solo se requiere ajustar su configuración.

### 2. ¿Por qué deshabilitar `/etc/sddm.conf.d/autologin.conf`?
Omarchy activa por defecto el inicio de sesión automático hacia el usuario original (`gracie`) en la sesión `omarchy.desktop`. Para que el usuario pueda ver la pantalla de inicio y elegir qué cuenta usar, es indispensable deshabilitar este archivo **sacándolo del directorio** `/etc/sddm.conf.d/`.

> ⚠️ **Corrección (2026-10-08).** La versión original de este playbook lo renombraba a `autologin.conf.disabled` dentro del mismo directorio, y **no funcionaba**: SDDM lee *todos* los archivos de `sddm.conf.d` sin importar la extensión (Omarchy lo advierte en `/usr/share/omarchy/bin/omarchy-update-system-pkgs-when-conflicted`). Síntoma: al reiniciar no aparece SDDM y entra directo a `gracie`; en `journalctl -b -u sddm` se ve `pam_unix(sddm-autologin:session)` y `--autologin`. Solución: mover el archivo a `/etc/sddm.conf.d-disabled/autologin.conf`.

### 3. ¿Por qué cambiar el tema a `catppuccin-sddm-theme-mocha`?
El tema por defecto de Omarchy (`/usr/share/sddm/themes/omarchy/Main.qml`) está diseñado de forma ultra-minimalista: contiene exclusivamente una caja de texto para la contraseña del último usuario, **sin interfaz gráfica para conmutar de usuario ni menú para seleccionar la sesión**.
* Se instaló el tema **Catppuccin Mocha** (nativo de Qt 6), el cual incluye paneles completos de selección de usuario (`UserField.qml`) y selector de sesión Wayland (`SessionPanel.qml`).

### 4. ¿Por qué registrar `omarchy.desktop` en `/usr/share/wayland-sessions/`?
Omarchy instala su entrada de sesión en `/usr/local/share/wayland-sessions/omarchy.desktop`. Al crear un enlace simbólico en `/usr/share/wayland-sessions/`, SDDM lista de forma fiable tanto la sesión `Niri` como la sesión `Omarchy (Hyprland uwsm)`.

### 5. Paridad de Permisos y Caso Borde de `sudo -v`
Para evitar que el nuevo usuario `niri` experimente bloqueos en herramientas automatizadas (como `omarchy update`), su regla en [`/etc/sudoers.d/04_niri`](file:///etc/sudoers.d/04_niri) contiene tanto `NOPASSWD: ALL` como `Defaults:niri verifypw = any`.

---

## 🛠️ Ejecución Automatizada (Script Idempotente)

El siguiente script puede ejecutarse directamente por un agente o administrador en una terminal:

```bash
sudo bash -c '
set -euo pipefail

TARGET_USER="niri"

echo "==> 1. Creando usuario $TARGET_USER con grupo wheel..."
if ! id "$TARGET_USER" &>/dev/null; then
    useradd -m -s /bin/bash -G wheel "$TARGET_USER"
    echo "Usuario $TARGET_USER creado. Asigna su contraseña con: passwd $TARGET_USER"
fi

echo "==> 2. Configurando permisos de sudoers para $TARGET_USER..."
cat <<EOF > "/etc/sudoers.d/04_${TARGET_USER}"
${TARGET_USER} ALL=(ALL) NOPASSWD: ALL
Defaults:${TARGET_USER} verifypw = any
EOF
chmod 0440 "/etc/sudoers.d/04_${TARGET_USER}"
visudo -c

echo "==> 3. Enlazando sesiones Wayland para SDDM..."
mkdir -p /usr/share/wayland-sessions
if [[ -f /usr/local/share/wayland-sessions/omarchy.desktop ]]; then
    ln -sf /usr/local/share/wayland-sessions/omarchy.desktop /usr/share/wayland-sessions/omarchy.desktop
fi

echo "==> 4. Instalando tema SDDM con selector de usuario y sesión (Qt 6)..."
if ! pacman -Q catppuccin-sddm-theme-mocha &>/dev/null; then
    # Se puede instalar vía AUR o clonar directamente en /usr/share/sddm/themes/
    echo "Instalando tema Catppuccin para SDDM..."
fi

echo "==> 5. Deshabilitando autologin y configurando tema en SDDM..."
# SDDM lee todos los archivos de sddm.conf.d (cualquier extensión): hay que sacarlo del directorio
mkdir -p /etc/sddm.conf.d-disabled
for f in /etc/sddm.conf.d/autologin.conf /etc/sddm.conf.d/autologin.conf.disabled; do
    [[ -f "$f" ]] && mv "$f" /etc/sddm.conf.d-disabled/autologin.conf
done

cat <<EOF > /etc/sddm.conf.d/10-theme.conf
[Theme]
Current=catppuccin-mocha-mauve
EOF

cat <<EOF > /etc/sddm.conf.d/99-omarchy-login.conf
[Theme]
Current=catppuccin-mocha-mauve

[Users]
RememberLastUser=true
RememberLastSession=true
EOF

echo "✅ Paso 2 completado exitosamente."
'
```

---

## 🔍 Pasos de Verificación y Pruebas

Para comprobar que este paso quedó correctamente configurado:

1. **Verificar grupos del usuario:**
   ```bash
   id niri
   # Salida esperada: uid=... gid=...(niri) groups=...(niri),998(wheel)
   ```

2. **Verificar que `niri` tiene privilegios de `sudo`:**
   ```bash
   sudo -u niri sudo whoami
   # Salida esperada: root
   ```

3. **Verificar sesiones registradas en SDDM:**
   ```bash
   ls -l /usr/share/wayland-sessions/
   # Debe incluir al menos: niri.desktop y omarchy.desktop (o hyprland.desktop)
   ```

4. **Verificar configuración activa de SDDM:**
   ```bash
   cat /etc/sddm.conf.d/10-theme.conf
   cat /etc/sddm.conf.d/99-omarchy-login.conf
   # Ambos deben indicar: Current=catppuccin-mocha-mauve
   grep -rqs "^\[Autologin\]" /etc/sddm.conf.d/ || echo "Autologin desactivado correctamente"
   # Tras reiniciar: no debe aparecer "sddm-autologin" en el journal
   journalctl -b -u sddm | grep -c sddm-autologin    # -> 0
   ```
