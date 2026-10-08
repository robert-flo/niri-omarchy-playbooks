# Fuentes externas clonadas en `~/Q/` (no versionadas aquí)

Estos directorios son clones de repos upstream con su propio historial git; están en `.gitignore`. Commits usados en esta instalación (2026-10-08):

| Directorio | Repo | Rama | Commit | Fecha del commit |
|---|---|---|---|---|
| `niri-edge/` | [StatIndet/niri-edge](https://github.com/StatIndet/niri-edge) | `main` | `b2aaeb3c` | 2026-10-06 |
| `quickshell/` (Clavis) | [StatIndet/quickshell](https://github.com/StatIndet/quickshell) | `main` | `c3ee54f` | 2026-10-07 |
| `key-cli/` | [StatIndet/key-cli](https://github.com/StatIndet/key-cli) | `main` | `3aed1bd` | 2026-09-28 |

Recrearlos en el mismo estado:

```bash
cd ~/Q
git clone https://github.com/StatIndet/niri-edge.git  && git -C niri-edge  checkout b2aaeb3c
git clone https://github.com/StatIndet/quickshell.git && git -C quickshell checkout c3ee54f
git clone https://github.com/StatIndet/key-cli.git    && git -C key-cli    checkout 3aed1bd
```

Compilación e instalación: `playbook-paso-1-niri-edge-y-key-cli.md` y `playbook-paso-3-clavis-shell-y-actualizaciones.md`.
