# Ansible Conventions

Mechanical, line-by-line conventions for the role code in this library. The
structural standards (namespacing, variable ownership, role siloing, lifecycle)
live in [STANDARDS.md](STANDARDS.md).

These are enforced across all roles and must be followed:

- **Task names**: all lowercase, always — no capitalization of any kind.
- **Booleans**: `true`/`false`, never `yes`/`no`.
- **FQCN**: always fully qualified (`ansible.builtin.copy`, `community.docker.docker_container`,
  not bare `copy`).
- **Imports**: `ansible.builtin.import_tasks` for static includes; `include_tasks` only
  when the include must be dynamic (e.g. dispatching on `os_family`).
- **Loops**: `loop`, not `with_items`.
- **File modes**: quoted strings (`"0644"`), never bare octals.
- **Variable namespacing**: every role variable prefixed with the role name
  (`docker_data_folder`, `<role>_<thing>`). Internal registered vars use a double
  underscore (`<role>__check`). See [STANDARDS.md §1.1](STANDARDS.md#11-namespacing).
- **Defaults**: every variable a role reads has a documented default in that role's
  `defaults/main.yml` — a reader learns the role's full surface from that file alone.
- **Jinja2 spacing**: `{{ var }}` with spaces, not `{{var}}`.
- **`become`**: never set inside a role; the consuming play sets it at the play level.
- **Handler names**: all lowercase, descriptive (`restart backup timer`,
  `update ca certificates`).
- **Check-mode safety on unprovisioned hosts**: a task that cannot work under
  `--check` on a not-yet-provisioned host — installing a package from a repo the
  same run adds, or acting on a service/group/binary an earlier real-only step
  creates (e.g. `community.docker.docker_compose_v2` before the docker engine
  exists, a `systemd`/`service` start before the unit is written, a supplementary
  group before it is created) — must be gated so a dry run skips it instead of
  erroring. Detect the prerequisite with a role-local `stat`
  (`register: <role>__<thing>`, `changed_when: false`), then guard the task —
  **and every handler it notifies**, since handlers fire under `--check` too —
  with `when: not ansible_check_mode or <role>__<thing>.stat.exists`. Real runs,
  and check runs against an already-provisioned host, are unaffected; only the
  un-previewable bare-host case is skipped. Keep the detect role-local rather than
  borrowing another role's registered fact — a role must stay runnable on its own
  (see role siloing in [STANDARDS.md](STANDARDS.md)).
- **Secrets**: a role reads a public, role-namespaced var (e.g. `gitea_postgres_password`);
  the consumer wires it to a `vault_`-prefixed value. Any task that handles secret
  material sets `no_log: true`. Roles never contain secret values.
