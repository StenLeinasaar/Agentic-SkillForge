# Ansible Debugging Playbooks

Structured failure-pattern trees by symptom category. Use these to pick the *next* diagnostic command quickly — but still present one command at a time per the protocol in `SKILL.md`, don't dump a tree on the user.

## Connectivity / SSH ("UNREACHABLE!")

Order of likelihood, cheapest check first:

1. **Basic reachability** — `ansible <host> -i <inventory> -m ping`. Rules in/out network vs. Ansible-specific issues in one shot.
2. **SSH auth** — if ping fails with an auth error: check `ansible_user`/`ansible_ssh_private_key_file` resolve correctly for that host (`ansible-inventory -i <inventory> --host <host>`), then try a raw `ssh -i <key> <user>@<host>` outside Ansible to isolate Ansible config from SSH itself.
3. **Host key / known_hosts** — errors mentioning host key verification: either the host was rebuilt (stale `known_hosts` entry) or `ansible_host` points somewhere unexpected. Don't blanket-disable `StrictHostKeyChecking` as a fix; confirm the host identity first.
4. **Python interpreter missing** — `/usr/bin/python: not found` or similar: target has no Python, or `ansible_python_interpreter` is misconfigured (common on minimal container images, network devices, or `python3`-only distros). Check with `ansible <host> -m raw -a "which python3"`.
5. **Timeouts vs. explicit refusal** — a hang-then-timeout usually means network/firewall/security-group; an immediate "connection refused" usually means the SSH daemon isn't listening on that port/host.

## Privilege escalation (`become`)

1. **Confirm become is even engaged** — check the task/play has `become: true` and the right `become_method`/`become_user`; a task silently running as the connection user is a common cause of "permission denied" that looks like a become failure but isn't.
2. **Password vs. NOPASSWD** — `sudo: a password is required` means the become method needs `--ask-become-pass` / `ansible_become_password`, or the account isn't in a NOPASSWD sudoers rule. Check `sudo -l` as that user directly on the host if you have another way in.
3. **become_method mismatch** — Windows targets need `runas`, not `sudo`; some appliances need `enable`/`su`. Confirm `ansible_connection` and OS family before assuming `sudo` is right.
4. **Escalation succeeds but the task still fails** — the *become* worked; the *task* itself is now failing as root/target user for an unrelated reason (permissions on a specific path, SELinux context, etc.). Don't keep pulling on the become thread once whoami/id confirms escalation worked.

## Module / task errors

1. **Read the actual `msg` field first** — most module failures return a specific `msg` explaining what went wrong; don't jump to `-vvv` before reading what's already in the output.
2. **Check module argument validity** — `Unsupported parameters for (module) module` means a typo'd or renamed parameter, often after a collection version bump. Check the module's docs for the installed collection version, not assumed-latest.
3. **"changed": false but task still marked failed** — distinguish a module-reported failure (bad state) from a task-level assertion (`failed_when`) firing on otherwise-successful output. Check if `failed_when` is set on the task before assuming the module itself is broken.
4. **Idempotency false positives** — a task reporting `changed: true` every single run when nothing actually changed: check whether it's `command`/`shell` (which is always "changed" unless `changed_when` is set) instead of a proper module, or whether the module's own idempotency check is being defeated by non-deterministic input (timestamps, generated IDs, unsorted lists).
5. **Version-dependent behavior** — if the same playbook behaves differently across environments, check `ansible --version` and `ansible-galaxy collection list` on both control nodes before assuming the target hosts differ.

## Jinja2 / templating errors

1. **`'dict object' has no attribute 'X'`** — almost always a variable that doesn't exist in that scope for that host: check variable precedence (host_vars > group_vars > role defaults > play vars — see precedence order below) with `ansible-inventory -i <inventory> --host <host> --vars`.
2. **Undefined variable only on some hosts** — the variable is set conditionally somewhere (host_vars file that doesn't exist for this host, a `set_fact` gated behind a `when` that didn't fire for this host). Trace where it's supposed to be set, not just where it's used.
3. **Template renders but with wrong value** — check for shadowing: the same variable name defined at multiple precedence levels. `ansible-inventory --host <host> --vars` shows the final resolved value; if it's not what you expect, something is overriding it "above" where you're looking.
4. **Filter/test errors** (`No filter named 'X'`) — usually a missing collection that provides that filter plugin; check `requirements.yml` / `ansible-galaxy collection list` for the collection providing it.
5. **Whitespace/newline issues in rendered files** — check `trim_blocks`/`lstrip_blocks` template config and `{%- -%}` whitespace-control syntax before assuming the logic itself is wrong.

## Variable precedence (when "it's set, so why isn't it applying")

From lowest to highest (abbreviated — full order is version-dependent, confirm against the installed Ansible-core docs when it matters):

```
role defaults  →  inventory group_vars  →  inventory host_vars  →
playbook group_vars/host_vars  →  host facts  →  play vars  →
role vars  →  block vars  →  task vars  →  set_fact / registered vars  →
extra vars (-e, always wins)
```

When a variable isn't taking effect, find where it's set at each relevant level with `ansible-inventory --host <host> --vars`, then check for an `-e` on the command line (or in AWX/AAP survey/extra vars) — it silently overrides everything.

## Handlers / notify

1. **Handler never fires** — the notifying task must report `changed`; a task that's a no-op (nothing actually changed) won't trigger it. Confirm the task actually changed with `-v` output.
2. **Handler name mismatch** — `notify:` must match the handler's `name:` exactly (or a `listen:` topic). A typo produces no error by default in older Ansible-core — it just silently never runs. Grep for the exact string.
3. **Handlers run at the end of the play, not immediately** — if a subsequent task in the same play depends on the handler having already run, either use `meta: flush_handlers` or restructure so the dependency isn't mid-play.
4. **Handler runs but the effect isn't visible** — could be running on the wrong host (handlers are per-host, notified only on hosts where the notifying task changed) — check `--limit` and the play's `hosts:` pattern.

## Loops / `with_items` / `loop`

1. **Loop var shadowing** — nested loops using the default `item` name in an included task silently shadow the outer loop's `item`. Use `loop_control: loop_var:` to rename.
2. **Loop over a dict vs. list mismatch** — `dict2items`/`items2dict` confusion is the most common cause of "loop only ran once" or "loop item has wrong shape."
3. **`with_items` flattening surprises** — legacy `with_items` flattens one level of nested lists automatically; `loop` does not. If migrating from `with_items` to `loop`, add `| flatten` explicitly if you relied on that behavior.

## Inventory / host pattern issues

1. **Host not in the play at all** — `ansible-inventory -i <inventory> --graph` to confirm the host is actually a member of the group(s) the play targets before debugging task logic.
2. **Dynamic inventory returning stale data** — check the inventory plugin's cache settings; a cached dynamic inventory can show hosts that no longer exist or miss newly created ones.
3. **Pattern excludes more than expected** — `--limit` combined with `hosts: group1:&group2` (intersection) or `!group3` (exclusion) syntax is a common source of "why didn't this run on host X" — check the effective host list with `ansible-playbook <playbook> --list-hosts` before assuming task logic is at fault.
