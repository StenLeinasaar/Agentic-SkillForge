# Ansible Best-Practices Checklist

Pull specific items from here during playbook/role reviews. Group findings by severity: correctness bug > security risk > idempotency gap > style.

## Security

- **Secrets never in plaintext.** No passwords, API keys, or tokens in `vars`, `group_vars`, `host_vars`, or committed defaults. Use `ansible-vault encrypt` / `encrypt_string`, or an external secrets backend via lookup plugins (HashiCorp Vault, AWS Secrets Manager, etc.).
- **Vault password not committed.** `.vault_pass` files or vault password scripts should be gitignored; CI/AWX should inject the vault password via a credential, not a checked-in file.
- **`no_log: true` on tasks that handle secrets** — otherwise the value leaks into logs/output even when the var itself is vaulted.
- **`become` scoped tightly.** Prefer task-level `become: true` over play-wide when only some tasks need root. Avoid `become_user: root` when a less-privileged user suffices.
- **Least-privilege connection accounts.** The SSH/WinRM account Ansible connects as shouldn't itself be root/admin unless there's a reason — escalate via `become` instead.
- **Pin collection and role versions** in `requirements.yml` (`version:` field). Unpinned `>=` or missing versions mean a `galaxy install` on a different day silently changes behavior.
- **Validate/sanitize any user-supplied input** that flows into `command`/`shell` — command injection risk is real when constructing shell strings from variables (survey inputs in AWX, extra vars, etc.). Prefer passing values as module parameters over string interpolation into a shell command.

## Idempotency

- **Prefer modules over `command`/`shell`.** A module can report accurate `changed` state and is idempotent by design; `command`/`shell` always reports `changed` unless you add `changed_when` (and often `creates`/`removes`).
- **Set `changed_when: false`** on read-only `command`/`shell` calls (status checks, version queries) so they don't falsely mark the play as having made changes.
- **Avoid non-deterministic input in tasks** — embedding timestamps, random values, or unsorted list/dict iteration order into a templated file will make idempotency checks (and diffs) noisy or wrong.
- **Test idempotency explicitly.** Running the same playbook twice should report zero `changed` on the second run. This is a fast, cheap sanity check that catches a large class of bugs — run it before considering a playbook "done."
- **Support `--check` / `--diff`.** Tasks using `command`/`shell` or `raw` don't support check mode by default (`check_mode: no` implicitly) — flag this in reviews, since it silently breaks `--check` runs.

## Structure & maintainability

- **Roles over ad-hoc task sprawl** once tasks are reused across playbooks or a play crosses ~50-100 lines. A role gives a stable interface (`defaults/main.yml`, `vars/main.yml`) instead of scattered `include_tasks`.
- **`defaults/` vs `vars/` distinction matters.** `defaults` = low-precedence, meant to be overridden by the caller. `vars` = higher-precedence, meant to be role-internal constants. Putting caller-facing knobs in `vars/main.yml` means callers can't easily override them without `-e`.
- **Tags on tasks/roles** that plausibly need selective execution (e.g., `--tags config` to push config changes without a full re-provision). Don't tag everything — tag where partial runs are a real use case.
- **`block`/`rescue`/`always` for error handling** instead of `ignore_errors: true`. `ignore_errors` masks failures silently; `rescue` lets you react to a specific failure and still fail loudly if the recovery itself fails.
- **Meaningful task `name:` fields.** Every task should have a `name:` describing intent (not restating the module call) — this is what shows up in `TASK [...]` output and is the first thing read during an incident.
- **Avoid deep `include_tasks`/`import_tasks` nesting** beyond 2-3 levels — it becomes hard to trace where a given task actually runs from during debugging.
- **`import_*` vs `include_*`**: `import_*` is static (resolved at parse time, supports tags cleanly, but no runtime loops/conditionals on the import itself); `include_*` is dynamic (supports runtime `when`/loop on the include, but tags and `--list-tasks` behave differently). Pick deliberately, not by habit.

## Inventory & variables

- **Group by function, not just environment.** `webservers`/`dbservers` groups with `env_prod`/`env_staging` as a cross-cutting group (via group-of-groups) scales better than `prod_webservers`/`staging_webservers` duplicated group trees.
- **Dynamic inventory for cloud-backed fleets** (`aws_ec2`, `azure_rm`, etc.) instead of hand-maintained static inventory files that drift from reality.
- **Document variable precedence assumptions** when a role relies on being overridden at a specific level — a comment or README note in the role saves the next person from re-deriving precedence order under pressure.
- **Avoid magic variable names colliding with Ansible internals** (`environment`, `hostvars`, etc.) — pick unambiguous names.

## Testing & CI

- **`ansible-lint` in CI**, not just locally — catches idempotency smells, deprecated syntax, and style issues before merge.
- **Molecule tests for roles**, at minimum a `converge` + idempotence scenario (run twice, assert no changes the second time). Roles used across many playbooks earn their test cost quickly.
- **Syntax check (`ansible-playbook --syntax-check`) and `--check --diff` dry runs** as a fast pre-merge gate before a full converge test.
- **Version-pin the CI runner's Ansible-core and collections** to match production execution environments — a CI pass on a different Ansible version than what actually runs the playbook is a false signal.

## Execution & operations

- **Use execution environments (or at least a pinned virtualenv)** rather than depending on whatever Ansible/collections happen to be installed on a given control node — reproducibility matters as much for Ansible itself as for the code it manages.
- **Limit blast radius on first run against new hosts** — `--limit` to a canary host or small batch, confirm, then roll out wider, rather than a fleet-wide run on unproven playbook changes.
- **`serial:` for rolling updates** on anything where taking all hosts down/degraded simultaneously is a problem (load-balanced app tiers, etc.).
- **Fact caching** for large inventories where `gathering: implicit` fact collection dominates runtime — weigh staleness risk against speed.
