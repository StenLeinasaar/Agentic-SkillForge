# Ansible Ecosystem Notes

Notes on collections/Galaxy, execution environments, AWX/Ansible Automation Platform, dynamic inventory, and Molecule — read the relevant section when a review or design question touches it rather than relying on memory for version-dependent specifics.

## Collections & Ansible Galaxy

- Since Ansible-core 2.10+, most modules live in **collections**, not the core package. `ansible` (the pip/package meta-package) bundles a curated set of community collections; `ansible-core` alone ships almost none. Know which one the user means when they say "Ansible version" — `ansible --version` shows both.
- **Pin collections in `requirements.yml`** with explicit `version:` constraints, and install via `ansible-galaxy collection install -r requirements.yml`. Unpinned installs drift silently across environments and time.
- **FQCN (fully-qualified collection name)** — modules should be referenced as `ansible.builtin.copy`, `community.general.ufw`, etc., not bare `copy`/`ufw`. Bare names rely on ambiguous resolution order across installed collections and can silently pick the wrong implementation if two collections provide a same-named module.
- **Private/internal collections** — organizations often publish internal collections to a private Galaxy instance (Automation Hub or a self-hosted Galaxy NG) for org-specific roles/modules. Check `ansible.cfg`'s `[galaxy]` section for non-default server URLs before assuming `galaxy.ansible.com` is the only source.
- **Collection vs. role distinction** — a collection can bundle roles, modules, plugins, and playbooks together; a standalone role (installed via `ansible-galaxy role install`) is an older, narrower unit. New shared automation is generally packaged as a collection now, not a bare role, unless there's a reason to keep it lightweight.

## Execution Environments (EE)

- An **execution environment** is a container image bundling a specific Ansible-core version, Python interpreter, and a pinned set of collections/dependencies — the modern answer to "works on my control node." Built with `ansible-builder` from a declarative `execution-environment.yml`.
- AWX/AAP run playbooks *inside* an EE by default (not on the control node's bare Python environment) — if a collection or Python dependency is missing at runtime in AWX but works locally, the EE image is almost always the first place to look, not the playbook.
- For local dev outside AWX, `ansible-navigator run --execution-environment true` runs against an EE image directly, catching "works on my machine, fails in AWX" mismatches earlier.
- Building EEs requires care around base image choice (Red Hat's `ee-minimal`/`ee-supported` vs. community base images) and keeping the collection/Python dependency lockfile in sync — treat `execution-environment.yml` and its generated lockfiles as part of the reviewed codebase, not a throwaway build artifact.

## AWX / Ansible Automation Platform (AAP)

- AWX is the open-source upstream project; **Ansible Automation Platform** is Red Hat's supported downstream product (also including EE tooling, private automation hub, event-driven Ansible). Behavior/version lag exists between the two — confirm which one the user is on before citing version-specific UI/API details.
- **Job templates** wrap a playbook + inventory + credentials + extra vars into a reusable, permissioned, auditable unit — this is usually the "unit of automation" in an AWX-managed org, not the raw playbook file.
- **Surveys** collect user input at launch time and inject it as extra vars — remember extra vars are the *highest* precedence, so a survey value silently overrides anything set in group_vars/role defaults. This is a common source of "why didn't my default take effect" confusion when debugging AWX-launched runs.
- **Credentials** are AWX-managed and injected at runtime (SSH keys, vault passwords, cloud API creds) — they generally shouldn't appear in inventory files or vars when AWX-launched; a hardcoded credential alongside AWX credential management is a smell worth flagging.
- **Workflow templates** chain job templates (including conditional branches on success/failure) — reach for these instead of a single mega-playbook when orchestrating multi-stage rollouts across different inventories/credentials.
- **Event-Driven Ansible (EDA)** reacts to external events (webhooks, message queues, monitoring alerts) and triggers rulebooks/playbooks automatically — relevant when the user describes wanting automation triggered by an external condition rather than run on a schedule or manually.

## Dynamic inventory

- Inventory plugins (`aws_ec2`, `azure_rm`, `gcp_compute`, `kubernetes.core.k8s`, etc.) query the source of truth live instead of a hand-maintained static file — the right default for any cloud-backed or frequently-changing fleet.
- **Caching** — most plugins support a `cache: true` setting with a TTL; know that a cached inventory can serve stale host lists, which shows up as "why isn't my new host in this play" or "why is this decommissioned host still targeted."
- **Grouping via `keyed_groups`/`compose`** in the inventory plugin config lets you derive Ansible groups from cloud tags/metadata (e.g., group by an `environment` tag) without maintaining a parallel static grouping — prefer this over post-hoc `group_by` tasks when the grouping key already exists as metadata.
- Mixing static and dynamic inventory sources is supported (multiple `-i` args or an inventory directory) — useful when some hosts are cloud-managed and others are fixed/on-prem.

## Molecule testing

- Molecule is the standard framework for testing roles/collections in isolation, typically against Docker or Podman containers (or cloud instances via delegated drivers) rather than real fleet hosts.
- A minimal useful scenario: **converge** (run the role) + **idempotence** (run it again, assert zero `changed`) + optionally **verify** (assertions via `ansible.builtin.assert` or a testinfra/pytest suite) for behavior beyond idempotency.
- Molecule tests a role/collection in isolation from the rest of the codebase's inventory and variable layering — a role can pass Molecule and still fail in the real playbook if the real inventory supplies different variable precedence than the test's `molecule.yml` `provisioner.inventory` does. Don't treat a green Molecule run as proof the integrated playbook works; it proves the role's own contract holds.
- CI should run Molecule per-role (or per-collection) on PRs touching that role, not as a single monolithic end-to-end test — keeps feedback fast and isolates which unit broke.
