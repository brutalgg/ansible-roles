# Ansible Role Standards

How the roles in this library are structured so they stay siloed, reusable, and
independent of any one environment. This is the authoritative reference for role
*structure* (namespacing, who owns what, lifecycle). The mechanical line-by-line
conventions (task naming, FQCN, booleans, file modes, register prefixes) live in
[CONVENTIONS.md](CONVENTIONS.md).

Scope: this library owns role source only. Inventory, playbooks, `group_vars`/
`host_vars`, and vault material belong to the consuming control repos and are out
of scope here. When a rule and existing code disagree, the rule wins — track the
gaps in the README.

## 1. Variable standards

### 1.1 Namespacing

- Every role variable is prefixed with the role name: `<role>_<thing>` (e.g.
  `docker_data_folder`, `caddy_sites`).
- Internal / registered variables use a **double underscore**: `<role>__<thing>`. These
  are role-private and must never be set from outside the role.
- Every variable a role reads has a documented default in its `defaults/main.yml`. A
  reader should learn the full surface of a role from its defaults file alone.

### 1.2 Variable ownership across roles (interface vars)

This is the rule that keeps roles independently reusable:

> A role reads and writes **only its own namespaced variables**. A role's tasks and
> templates must never reference another role's variables.

When role A needs data that conceptually belongs to domain B, B does not reach into A —
instead, the consumer maps its own values into A's documented **interface variable**.
Data flows *into* the consumer through the consumer's namespace.

Concretely, wiring a reverse proxy in front of an app: a generic `proxy` role exposes an
interface var, and the app's environment populates it:

```yaml
# roles/proxy/defaults/main.yml  — proxy owns the interface, knows nothing about any app
proxy_sites: []   # list of { name, domains: [...], upstream }

# (in the consuming control repo) — app maps its own vars into proxy's interface
proxy_enabled: true
proxy_sites:
  - name: app
    domains: "{{ app_domains }}"
    upstream: "127.0.0.1:{{ app_listen_port }}"
```

The anti-pattern is a role's template referencing another role's variables directly
(e.g. the `proxy` role reading `app_domains`). That couples two roles that should not
know about each other and makes the role un-reusable. A single `proxy_enabled` guard —
owned by `proxy`, set by consumers — is the on/off switch.

### 1.3 Secrets

- A role reads a public, role-namespaced variable (e.g. `<role>_secret`); the consuming
  control repo wires that to a `vault_`-prefixed value from its own encrypted vault.
- Roles never contain secret values, and never assume a particular vault layout.
- Tasks that handle secret material set `no_log: true`.

## 2. Role siloing

- **Single responsibility.** A role manages one concern (the engine, the trust store, the
  backups). If a role grows a second unrelated concern, split it.
- **Self-contained.** A role's behavior is fully described by its `defaults` + `tasks` +
  `templates` + `handlers`. It reads only its own namespace ([1.2](#12-variable-ownership-across-roles-interface-vars))
  and declares a default for everything it reads.
- **Standard layout:**
  - `tasks/main.yml` is a thin orchestrator that `import_tasks` the sub-task files
    (`install.yml`, `config.yml`, …) in order. Keep real logic in the sub-files.
  - `defaults/main.yml` — documented variables.
  - `handlers/main.yml` — lowercase, descriptive handler names.
  - `templates/` — `.j2` files; reference only this role's namespace + facts.
- **New role vs. extend?** Add a new role when the concern is independently deployable or
  reusable. Extend an existing role when the work is a variation of its existing single
  responsibility. Prefer an interface variable ([1.2](#12-variable-ownership-across-roles-interface-vars))
  over a cross-role reference whenever two roles need to cooperate.

### 2.1 Lifecycle functions

A role manages the **whole life** of its concern, not just first install. Split the
lifecycle into discrete, individually-runnable, idempotent task files so each function can
be exercised on its own. `tasks/main.yml` imports only the **provisioning** subset
(install + config + conditional tls); out-of-band functions (update, teardown) live in
their own files that `main.yml` does **not** import — they are reached through the
consumer's playbook layer instead.

Canonical task files (a role need not have all):

| File | When it runs | Notes |
|---|---|---|
| `install.yml` | provisioning | Bring the service into existence. Guard one-time bootstrap (binary/sentinel `stat` check) so re-runs are no-ops; package installs use `state: present`. |
| `config.yml` | provisioning | Re-applied host-local configuration. Idempotent; the steady-state desired config. |
| `tls.yml` | provisioning, conditional | Cert material, gated on a `<role>_tls_enabled`-style flag. |
| `update.yml` | **out of band only** | Upgrades to a newer release (`state: latest`, vendor self-update). **Never** imported by `main.yml` — a provisioning run must not silently upgrade a service. The consumer runs it via an operations playbook (`import_role … tasks_from: update.yml`). |

Rules:

- **Updates are opt-in, never incidental.** Provisioning installs and converges config; it
  does not upgrade versions. Keep the upgrade in `update.yml` and let the consumer invoke
  it deliberately.
- **One function per file, idempotent.** Each file is safe to run repeatedly and in
  isolation. A reader maps a lifecycle question ("how do I upgrade this?", "how do I rotate
  its cert?") to exactly one file.
- **Decide deliberately what a role does *not* manage.** Some functions belong to the
  platform or the container, not a role. State the boundary in the role's defaults/comments
  rather than half-implementing it.

### 2.2 Multi-distro roles

A role that runs across OS families keeps the **distro-agnostic logic shared** and
quarantines only the parts that genuinely differ — package/repository management, service
names, a few naming conventions. Lean on Ansible's abstraction modules (`package`, `user`,
`service`, `hostname`, `lineinfile`) rather than branching by distro.

Pattern (the `base` OS-baseline role is the reference):

- **`vars/<os_family>.yml`** holds the family-specific names the shared tasks read
  (`<role>__ssh_service`, …), loaded at the top of `tasks/main.yml` via `include_vars` +
  `with_first_found` on `distribution` → `os_family` (tagged `always`). These are
  role-private (`__`): set by the role from facts, never from outside.
- **`tasks/<concern>-<os_family>.yml`** carries the per-family implementation of a divergent
  concern (`packages-Debian.yml` owns apt sources; a `packages-RedHat.yml` would own dnf).
  A thin `tasks/<concern>.yml` dispatches with `include_tasks` on `os_family`.
- **Generic installs use `ansible.builtin.package`**, not `apt`/`dnf` — reserve the
  manager-specific module for tasks that need its semantics (cache control, `upgrade`).
- **Sub-variants live inside their family.** A Debian respin stays *inside*
  `packages-Debian.yml`, not a separate family.

Adding a distro is then additive: drop in `vars/<family>.yml` +
`tasks/<concern>-<family>.yml`; the shared tasks are untouched.

## 3. Linting

Run before committing:

```bash
ansible-lint && yamllint .
```

- `.ansible-lint` skips `name[casing]` (all-lowercase task names are intentional).
- `.yamllint` allows long lines where unavoidable (SSH keys, signed-by repo URLs).
- The whole library passes `ansible-lint` at the **production** profile.
