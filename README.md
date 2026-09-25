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
| `sync_repo` | Repo-Stand auf den Zielhost bringen (`git`: Host zieht `repo_branch`; `push`: Controller schiebt seinen Checkout) | `repo_dest`, im `git`-Modus `repo_url` |
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
| `repo_sync_mode` | `git` | `git`: Zielhost zieht selbst. `push`: Controller schiebt per `git archive`, Zielhost braucht keinen Git-Server (s. u.) |
| `repo_branch` | `main` | Auszucheckender Ref im `git`-Modus (Override: `-e repo_branch=…`) |
| `repo_src_dir` / `repo_push_ref` | `{{ playbook_dir }}` / `HEAD` | `push`: Controller-Checkout und zu deployender Ref |
| `repo_push_allow_dirty` | `false` | `push`: uncommittete Änderungen an getrackten Dateien tolerieren (werden **nicht** mit deployt) |
| `repo_key_file` / `repo_become_user` | omit | optionaler Deploy-Key / git-User |
| `env_in_repo` | `true` | `.env` in `<stack>/.env` (homelab) vs. `services_config_root/<n>/<n>.env` (hetzner) |
| `services_config_root` | `/opt/services-config` | Basis für absolute Config (nur `env_in_repo=false`) |
| `config_owner` / `config_group` | `root` | Owner gerenderter Configs/Dirs (homelab: `simon`) |
| `manifest_templates_dir` | `{{ playbook_dir }}/templates` | absoluter Pfad zu den `*.j2` |
| `ntfy_enabled` / `ntfy_user` | `false` / `admin` | ntfy an/aus + Basic-Auth-User |

## `push`-Modus

Für Hosts, die den Git-Server selbst betreiben (Forgejo auf dem VPS): Zöge der Host
seinen Code von dort, könnte ein Deploy, der Forgejo zerlegt, seinen Fix nicht mehr
holen. Im `push`-Modus baut der Controller `git archive` aus seinem Checkout, entpackt
es nach `<repo_dest>.new` und tauscht das Verzeichnis aus — auf dem Host liegt eine
Arbeitskopie **ohne `.git`**, plus Marker `<repo_dest>/.deploy-rev` (`commit`, `tree`).
Gleicher Tree → kein Transfer. Deployt wird nur Committetes; ein unsauberer Checkout
bricht ab. Voraussetzung: in `repo_dest` liegt nichts Laufzeitrelevantes (der Tausch
löscht es), Configs/Daten gehören nach `services_config_root`.

## Versionierung

Änderungen: bearbeiten → **neuen Tag** (`vX.Y.Z`) → `version:` in beiden Repos bumpen →
beide testen. **Nicht** auf `main` pinnen, sonst schlägt ein Commit ungetestet in beide
Auto-Deploys durch.
