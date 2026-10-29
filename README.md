# iq-playbooks

Diagnostic playbooks that IQ Core runs through Semaphore when a Zabbix alert raises a
Zammad ticket. IQ Core picks the playbook for the alert, Semaphore runs it over SSH on the
one host in the alert, and the output is pasted into the ticket as an internal note.

Whatever is on `main` is what runs, so every rule below matters.

## Rules for every playbook

1. **Read-only.** Look, never change: no restarts, installs, deletes, file edits or config
   changes. Every task that runs a command gets `changed_when: false`.
2. **No sudo.** Never use `become`. Playbooks run as `iqdiag`, which has no sudo, so a task
   that needs root will fail.
3. **One host only.** Use `hosts: all` and keep the first task from
   `diag_top_processes.yml` that refuses to run on more than one host. Semaphore narrows
   the run to the alert's host with its Limit prompt.
4. **No secrets in output.** The output goes into a ticket many people read. Print process
   names (`comm`), never full command lines (`args`), and never print environment
   variables, config files or anything from `/etc/shadow`, `.env` and similar.
5. **Short output.** End with one `debug` task named `Report` that prints only what an
   engineer needs, trimmed (for example the top 5 lines, not the whole list).
6. **Fast.** Set `timeout: 30` at play level and `gather_facts: false` unless you need facts.
7. **Linux first.** Windows (WinRM) and Cisco need different modules; ask before writing one.

## Adding one

1. **Add file** → **Create new file** (or **Upload files**), name it `diag_<what>.yml`, commit.
2. Ask a reviewer to check it against the rules above.
3. A Semaphore admin adds a template for it. Until then IQ Core cannot run it.

## Template to copy

```yaml
- name: <What this checks>
  hosts: all
  gather_facts: false
  any_errors_fatal: true
  timeout: 30
  tasks:
    - name: Refuse to run on more than one host
      ansible.builtin.assert:
        that: ansible_play_hosts_all | length == 1
        fail_msg: "Run with --limit set to exactly one host."

    - name: <First check>
      ansible.builtin.command: <read-only command>
      register: first
      changed_when: false

    - name: Report
      ansible.builtin.debug:
        msg:
          first: "{{ first.stdout_lines[:10] }}"
```
