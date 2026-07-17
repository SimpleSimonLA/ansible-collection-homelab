# homelab.stacks

Geteilte, **manifest-getriebene** Deploy-Logik für die Repos `homelab` und
`hetzner-vps`. Enthält die Rolle **`manifest_deploy`** mit vier wiederverwendbaren
Task-Einstiegspunkten, die vorher in beiden Repos dupliziert waren.

## Installation

In der `collections/requirements.yml` des jeweiligen Repos:

```yaml
collections:
  - name: git+https://github.com/SimpleSimonLA/ansible-collection-homelab.git
    type: git
    version: v0.2.0    # auf Tag pinnen (nicht main)
```

Dann (Controller-seitig; Semaphore macht das als Vorstufe):

```bash
ansible-galaxy collection install -r collections/requirements.yml
```

## Rolle `manifest_deploy` — Task-Einstiegspunkte

Aufruf via `include_role` mit `tasks_from`:

| `tasks_from` | Zweck | Setzt / erwartet |
|---|---|---|
| `sync_repo` | Repo auf dem Zielhost auf `repo_branch` ziehen | `repo_url`, `repo_dest` |
| `discover_stacks` | Manifeste (`stacks/<n>/deploy.meta.yml`) einsammeln | `stacks_root` → `managed_stacks`, `managed_stack_names` |
| `prepare_stacks` | `.env`/Config rendern, Data-Dirs anlegen | `managed_stacks`, Vault |
| `notify_ntfy` | ntfy-Notification senden | `ntfy_enabled`, `ntfy_url`, `ntfy_password` + `ntfy_title`/`ntfy_body` |

```yaml
- name: Discover managed stacks
  ansible.builtin.include_role:
    name: homelab.stacks.manifest_deploy
    tasks_from: discover_stacks
  vars:
    stacks_root: "{{ my_stacks_root }}"
```

## Parametrisierung (Defaults in `roles/manifest_deploy/defaults/main.yml`)

| Var | Default | Zweck |
|---|---|---|
| `repo_branch` | `main` | Auszucheckender Ref (Override: `-e repo_branch=…`) |
| `repo_key_file` / `repo_become_user` | omit | optionaler Deploy-Key / git-User |
| `env_in_repo` | `true` | `.env` in `<stack>/.env` (homelab) vs. `services_config_root/<n>/<n>.env` (hetzner) |
| `services_config_root` | `/opt/services-config` | Basis für absolute Config (nur `env_in_repo=false`) |
| `config_owner` / `config_group` | `root` | Owner gerenderter Configs/Dirs (homelab: `simon`) |
| `manifest_templates_dir` | `{{ playbook_dir }}/templates` | absoluter Pfad zu den `*.j2` |
| `ntfy_enabled` / `ntfy_user` | `false` / `admin` | ntfy an/aus + Basic-Auth-User |

## Versionierung

Änderungen: bearbeiten → **neuen Tag** (`vX.Y.Z`) → `version:` in beiden Repos bumpen →
beide testen. **Nicht** auf `main` pinnen, sonst schlägt ein Commit ungetestet in beide
Auto-Deploys durch.
