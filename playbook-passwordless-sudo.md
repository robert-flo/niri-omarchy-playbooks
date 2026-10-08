# Playbook: Configuración Permanente de Sudo sin Contraseña (Passwordless Sudo)

Este documento es una guía paso a paso y playbook automatizado diseñado para ser ejecutado por un usuario o suministrado directamente a un agente de Inteligencia Artificial (ej. Antigravity, Claude Code, Cursor, Aider, etc.) en un nuevo equipo con Linux (especialmente optimizado para **Omarchy / Arch Linux** y distribuciones basadas en systemd).

---

## 🎯 Objetivo

Configurar el usuario actual para que tenga privilegios completos de `sudo` **sin solicitar contraseña de forma permanente**, evitando temporizadores automáticos y resolviendo problemas de scripts automatizados que usan `sudo -v` (como `omarchy update`).

---

## ⚠️ Problema y Caso Borde Resuelto

En sistemas modernos como Omarchy o Arch Linux:
1. **Reglas temporales:** Herramientas como `omarchy-sudo-passwordless` crean reglas temporales con temporizadores de `systemd` que expiran en 15 minutos.
2. **El bug de `sudo -v` (`verifypw`):** Simplemente poner `USUARIO ALL=(ALL) NOPASSWD: ALL` **no es suficiente** si existen otros archivos en `/etc/sudoers.d/` (como `50-asdcontrol`) que tienen reglas sin `NOPASSWD`. La directiva por defecto de sudo es `verifypw = all`, lo que causa que herramientas que validan credenciales con `sudo -v` (como `omarchy-update-stay-awake`) sigan pidiendo contraseña.
3. **Solución completa:** Se requiere tanto la regla de bypass como la directiva `Defaults:<USUARIO> verifypw = any`.

---

## 🤖 Prompt / Instrucción para el Agente

Si vas a darle esta tarea a un agente IA en una nueva PC, puedes copiarle directamente el siguiente prompt:

```text
Configura mi usuario actual para que tenga sudo permanente sin contraseña (passwordless sudo).
Sigue estas pautas estrictas:
1. Detecta mi usuario actual dinámicamente con $USER (o whoami).
2. Crea o actualiza el archivo en /etc/sudoers.d/ (por ejemplo /etc/sudoers.d/04_$USER o /etc/sudoers.d/99-$USER-nopasswd).
3. Asegúrate de incluir AMBAS líneas para evitar bloqueos con 'sudo -v':
   <USUARIO> ALL=(ALL) NOPASSWD: ALL
   Defaults:<USUARIO> verifypw = any
4. Asigna permisos 0440 al archivo.
5. Valida la sintaxis con 'sudo visudo -c'.
6. Comprueba que 'sudo -k && sudo -n whoami' y 'sudo -n -v' funcionen con código de salida 0.
```

---

## 🛠️ Ejecución Automatizada (Script Bash Idempotente)

El agente o el usuario pueden ejecutar el siguiente comando seguro de una sola línea en la terminal:

```bash
sudo bash -c '
set -euo pipefail

TARGET_USER="${SUDO_USER:-$USER}"
FILE="/etc/sudoers.d/04_${TARGET_USER}"

echo "==> Configurando passwordless sudo para: ${TARGET_USER}"

cat <<EOF > "$FILE"
${TARGET_USER} ALL=(ALL) NOPASSWD: ALL
Defaults:${TARGET_USER} verifypw = any
EOF

chmod 0440 "$FILE"

echo "==> Validando sintaxis con visudo..."
visudo -c

echo "==> Probando privilegios..."
sudo -u "$TARGET_USER" -k
sudo -u "$TARGET_USER" -n whoami > /dev/null
sudo -u "$TARGET_USER" -n -v > /dev/null

echo "✅ Configuración completada exitosamente y verificada."
'
```

---

## 🔍 Pasos de Verificación Manual

Para comprobar que todo quedó bien configurado:

1. **Verificar que el validador de credenciales pasa sin contraseña:**
   ```bash
   sudo -k && sudo -n -v
   ```
   *(Debe retornar código de salida 0 sin pedir contraseña)*

2. **Verificar que un comando administrativo se ejecuta como root:**
   ```bash
   sudo -n whoami
   ```
   *(Debe responder `root`)*

3. **Verificar sintaxis de todos los archivos de sudoers:**
   ```bash
   sudo visudo -c
   ```
   *(Todos los archivos deben figurar con `parsed OK`)*
