# iq-playbooks

Diagnostic playbooks for **IQ Core**. When a monitoring alert opens a helpdesk ticket, IQ Core
runs the matching playbook on the affected server and pastes the result into the ticket, so
the engineer who picks it up already has the diagnosis.

This README covers what IQ Core is, where it's heading, how the automation works, and
**how to write and add a playbook**. If you only want to add a playbook, jump to
[Writing a playbook](#writing-a-playbook).

> This repo is **public**. Never commit IP addresses, host names, user names, passwords,
> keys or tokens. Servers and credentials live in Semaphore, never here.

---

## Contents
1. [IQ Core](#iq-core)
2. [The harness plan](#the-harness-plan)
3. [How the automation works](#how-the-automation-works)
4. [Ansible and Semaphore in five minutes](#ansible-and-semaphore-in-five-minutes)
5. [The rules](#the-rules)
6. [Anatomy of a playbook](#anatomy-of-a-playbook)
7. [Writing a playbook](#writing-a-playbook)
8. [Adding it to this repo](#adding-it-to-this-repo)
9. [Reviewer checklist](#reviewer-checklist)
10. [Command cookbook](#command-cookbook)
11. [Template to copy](#template-to-copy)
12. [FAQ](#faq)
13. [Glossary](#glossary)

---

## IQ Core

IQ Core is our AI operations assistant. It connects to the systems we already run and
answers questions about them in plain language:

| System | What IQ Core reads from it |
|---|---|
| Zabbix | Network devices, Windows and Linux hosts: status, problems, metrics |
| Zammad | Helpdesk tickets |
| Prometheus | The Linux cluster, OpenStack VMs and containers: CPU, memory, disk, network |
| Ceph | Storage under OpenStack: capacity, pools, disks, snapshots |
| OpenSearch | OpenStack application logs |

It has two ways in:
- **Chat:** an engineer asks "which servers have high CPU?" and IQ Core queries the right
  system and answers.
- **Ticket assist:** when Zabbix opens a Zammad ticket, IQ Core adds an internal note with a
  suggested fix. This was IQ Core's first write to any system.

Until now IQ Core only **read** data and **wrote notes**. This repo is the first time it
**runs commands on servers**, which is why the rules below are strict.

## The harness plan

A language model on its own only turns text into text. A **harness** is the program around
it that makes it useful. Tools like Claude Code and Codex are harnesses:

1. **A loop:** give the model a task and a list of tools; it either answers or asks to use a
   tool; the harness runs the tool, hands back the result, and asks again.
2. **Tools:** functions the model may call (query Zabbix, read a ticket, run a playbook).
3. **Permissions:** before running a tool, the harness decides: run it, ask a human, or refuse.
4. **Triggers:** what starts a run (a chat message, a new ticket, a schedule).
5. **A record:** what ran, with what input, and what came back.

IQ Core already has the loop, the tools, the triggers and the record. What it is gaining now
is **permissions**, because its new tools *act* instead of only reading:

- Every tool is labelled **read** or **act**.
- Read tools run freely.
- Act tools run only when allowed: from chat, a human approves; from a ticket, only an
  approved list runs.

Running a diagnostic playbook is the **first act capability**. More will follow the same
pattern: a tool, a risk label, and an approved list.

**Phases**

| # | Phase | State |
|---|---|---|
| 0 | Ansible server (Semaphore) running | Done |
| 1 | Semaphore can log into target servers | First server done |
| 2 | Diagnostic playbooks in this repo, run by hand from Semaphore | In progress |
| 3 | IQ Core client: alert → playbook → Semaphore → output | Next |
| 4 | Dry run on real tickets for a week (logs what *would* run) | |
| 5 | Live: output written into tickets | |
| 6 | Harness permissions for every act tool; fix playbooks only with human approval | Later |

## How the automation works

```
 Zabbix alert ──► Zammad ticket ──► IQ Core
                                      │ 1. reads the problem and the server from the ticket
                                      │ 2. looks up the approved playbook for that problem
                                      ▼
                                  Semaphore ──► pulls the playbook from this repo (main)
                                      │
                                      │ 3. Ansible connects over SSH, as a user with no sudo,
                                      ▼    to that one server only, and runs the playbook
                                 Target server
                                      │
                                      ▼ 4. output
                                  Semaphore ──► IQ Core ──► internal note on the ticket
```

Important points:
- **The AI never writes or picks commands.** It only matches the alert to a playbook from a
  fixed list. Every command that runs is one *you* wrote and someone reviewed.
- **Whatever is on `main` is what runs.** Semaphore pulls the latest `main` on every run, so
  an edit to an existing playbook is live on its next run.
- **A new playbook is not live until a Semaphore admin creates a template for it.**

## Ansible and Semaphore in five minutes

**Ansible** runs commands on servers over SSH. You describe the steps in a YAML file (a
**playbook**); Ansible connects to each server and runs them. It is **agentless**: nothing is
installed on the target servers, and playbooks never live on them.

**Semaphore** (Semaphore UI) is a web app around Ansible. It stores the SSH keys (encrypted),
the list of servers, and the **templates** (a playbook plus the settings to run it). It keeps
every run's output, and it has an API, which is how IQ Core starts runs.

Words you'll see in Semaphore:

| Semaphore | Meaning |
|---|---|
| Project | A group of templates, keys and inventories. IQ Core can only run templates in its own project, `IQ Core diagnostics` |
| Key Store | Encrypted SSH keys Semaphore uses to log in |
| Repository | Where playbooks come from: this repo |
| Inventory | The servers Semaphore may connect to |
| Environment | Variables passed to every run |
| Task template | One playbook plus its inventory, repository and environment, ready to run |
| Task | One run of a template, with its full output |
| Limit | Narrows a run to one server. IQ Core always sets it |

## The rules

Every playbook in this repo must follow all of them. The reviewer checks each one.

1. **Read-only.** Look, never change. No restarts, installs, deletes, file writes, config
   changes or kills. Every command task gets `changed_when: false`.
2. **No sudo.** Never use `become`. Playbooks run as a user with no sudo, so anything that
   needs root fails. Write checks that work as a normal user.
3. **One server only.** Use `hosts: all` and keep the "Refuse to run on more than one host"
   task as the first task. Semaphore narrows the run with Limit; this task is the safety net
   if Limit is ever missing.
4. **No secrets in output.** The output is pasted into tickets that many people read.
   - Process names (`comm`), never full command lines (`args`, `ps aux`, `top -c`):
     command lines often contain passwords and tokens.
   - Never print environment variables, `.env` files, config files, or anything under
     `/etc/shadow`, `~/.ssh` or similar.
   - Nothing about servers in the playbook itself either: this repo is public.
5. **Short output.** End with exactly one `debug` task named `Report` that prints only what
   an engineer needs, trimmed (the top 5 or 10 lines, not hundreds).
6. **Fast.** `gather_facts: false` unless you need facts, `timeout: 30` at play level, and no
   command that runs longer than about 10 seconds.
7. **Linux only for now.** Windows (WinRM) and Cisco need different modules. Ask before
   writing one.
8. **Built-in modules only.** Use `ansible.builtin.*`. Don't depend on collections or tools
   the server may not have; if a command needs a package (like `iostat` from `sysstat`),
   check it's installed on the target servers first, because a playbook can't install it.

## Anatomy of a playbook

This is `diag_top_processes.yml`, the first playbook, piece by piece.

```yaml
- name: Top processes by CPU and memory   # shown in Semaphore's output
  hosts: all                              # Semaphore's Limit narrows this to one server
  gather_facts: false                     # skip Ansible's slow system survey
  any_errors_fatal: true                  # one failure stops the whole run
  timeout: 30                             # no task may run longer than 30 seconds
  tasks:
```

**The safety net.** Always first, always unchanged:

```yaml
    - name: Refuse to run on more than one host
      ansible.builtin.assert:
        that: ansible_play_hosts_all | length == 1
        fail_msg: "Run with --limit set to exactly one host."
```

**A check.** Run one read-only command and keep its output under a name (`register`):

```yaml
    - name: Memory
      ansible.builtin.command: free -m     # the command, exactly as you'd type it
      register: mem                        # its output is now available as mem.stdout
      changed_when: false                  # tells Ansible this changed nothing
```

**The report.** One `debug` task at the end prints what goes into the ticket:

```yaml
    - name: Report
      ansible.builtin.debug:
        msg:
          memory_mb: "{{ mem.stdout_lines }}"            # every line
          top_memory: "{{ by_mem.stdout_lines[:6] }}"    # header + top 5 only
```

What you can use from a registered result:

| Expression | Gives |
|---|---|
| `x.stdout` | The whole output as one string |
| `x.stdout_lines` | The output as a list of lines |
| `x.stdout_lines[:6]` | The first 6 lines |
| `x.stdout_lines[-10:]` | The last 10 lines |
| `x.rc` | The exit code (0 = success) |

**`command` versus `shell`.** Prefer `ansible.builtin.command`: it runs the program directly,
so `|`, `>`, `&&` and `*` do **not** work. If you truly need a pipe, use `ansible.builtin.shell`
with a fixed command string, and never build the command from a variable. Usually you can
avoid the pipe: run the command plain and trim with `[:6]` in the report.

**A command that may fail.** If a check can fail without the whole diagnosis being useless
(a log file that may not exist, a service that may not be installed), add
`failed_when: false` and report its `rc`:

```yaml
    - name: Is nginx running
      ansible.builtin.command: systemctl is-active nginx
      register: nginx
      changed_when: false
      failed_when: false        # inactive returns rc 3; still worth reporting
```

## Writing a playbook

1. **Pick the alert.** Start from a real Zabbix alert that keeps coming up, for example
   "Disk read/write request responses are too high" or "Load average is too high". Ask the
   team which alerts happen most: those get playbooks first.
2. **Write down what you'd check by hand.** If you were paged for this alert, what would you
   type in the first two minutes? Those commands are the playbook.
3. **Test each command as a normal user.** Log into a test server as a user *without* sudo and
   run each command. If it says "permission denied" or shows nothing useful, it won't work in
   the playbook either. Time it: each should finish in a few seconds.
4. **Check the output for secrets.** Would you be comfortable with everyone who reads tickets
   seeing it? If not, change the command (for example `comm` instead of `args`).
5. **Copy the [template](#template-to-copy)** and fill it in: one task per command, then the
   `Report`.
6. **Name the file** `diag_<what_it_checks>.yml`, lowercase with underscores:
   `diag_disk_io.yml`, `diag_memory.yml`, `diag_service_nginx.yml`.
7. **Name the tasks** in plain words ("Disk usage by mount", not "task2"). They appear in
   Semaphore's output.
8. **Write the header comment:** one line on what alert it's for and one on what it reports.

## Adding it to this repo

You don't need git on your machine; GitHub's website is enough.

1. Open this repo on GitHub → **Add file** → **Create new file** (or **Upload files** if you
   wrote it locally).
2. Name it `diag_<what>.yml` and paste the playbook.
3. Click **Commit changes**, choose **Create a new branch for this commit and start a pull
   request**, then **Propose changes**. This opens a pull request instead of changing `main`
   directly.
4. In the pull request, say which alert it's for and paste the output from your manual test.
5. A reviewer checks it against the [checklist](#reviewer-checklist) and merges it.
6. A Semaphore admin creates a template for it in the `IQ Core diagnostics` project (with
   the Limit prompt on) and runs it once on a test server.
7. The admin tells IQ Core which alert maps to the new playbook. Until then IQ Core won't
   run it.

**Editing an existing playbook** works the same way (open the file → pencil icon → propose
changes). Remember that once merged it is live on the next run.

## Reviewer checklist

- [ ] Every task is read-only, and every command task has `changed_when: false`
- [ ] No `become`, no `sudo` inside commands
- [ ] `hosts: all`, and the first task is the unchanged "Refuse to run on more than one host"
- [ ] `gather_facts: false` (or a reason in a comment) and `timeout: 30`
- [ ] Only `ansible.builtin.*` modules; `shell` only with a fixed string and a reason
- [ ] No full command lines, environment variables, config contents or credentials in output
- [ ] No IPs, host names, user names or secrets anywhere in the file
- [ ] Ends with a single `Report` task with trimmed output
- [ ] Every command was tested by hand as a user without sudo, and the PR shows that output
- [ ] File name is `diag_<what>.yml` with a header comment

## Command cookbook

Read-only commands that work without sudo on most Linux servers. Test on your targets first.

| Alert type | Command | Notes |
|---|---|---|
| High CPU / load | `uptime` | Load averages for 1, 5, 15 minutes |
| | `top -b -n 2 -d 1 -o %CPU -w 200` | Use the **second** sample; the first is averaged since boot. See `diag_top_processes.yml` for how to cut it |
| | `nproc` | CPU count, to judge whether a load is high |
| High memory | `free -m` | `available` is what matters, not `free` |
| | `ps -eo pid,user,%cpu,%mem,etime,comm --sort=-%mem` | Accurate for memory. **Not** for CPU: `ps` `%cpu` is a lifetime average |
| Disk full | `df -h` | Space per mount |
| | `df -i` | Inodes: a disk can be "full" with free space |
| Disk slow (I/O) | `cat /proc/pressure/io` | How much time tasks spent waiting on disk (kernel 4.20+) |
| | `iostat -x 1 2` | Per-disk latency. Needs `sysstat` installed; use the second sample |
| | `cat /proc/diskstats` | Raw per-disk counters, always available |
| Network | `ss -s` | Socket summary |
| | `ip -s link` | Errors and drops per interface |
| Service | `systemctl is-active <name>` | Add `failed_when: false`; inactive returns non-zero |
| | `systemctl status <name> --no-pager -n 0` | State without log lines |
| Recent reboot | `uptime -s` | When the server last started |
| | `last -x reboot -n 5` | Reboot history (needs `/var/log/wtmp` readable) |
| Time | `timedatectl` | Clock and sync state |

**Won't work without sudo** (avoid): `journalctl` for system logs (a normal user only sees
their own), `iotop`, `dmesg` on most distros, reading `/var/log/syslog` or `/var/log/messages`,
`ss -p` (process names of other users' sockets), `du` on other users' directories.

## Template to copy

```yaml
# For alert: <the Zabbix alert name this answers>
# Reports: <what the engineer will see>
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

    - name: <First check, in plain words>
      ansible.builtin.command: <read-only command>
      register: first
      changed_when: false

    - name: <Second check>
      ansible.builtin.command: <read-only command>
      register: second
      changed_when: false

    - name: Report
      ansible.builtin.debug:
        msg:
          first: "{{ first.stdout_lines[:10] }}"
          second: "{{ second.stdout_lines[:10] }}"
```

## FAQ

**Why not let playbooks fix things, like restarting a service?**
Not yet. A wrong restart at 3 a.m. is worse than no restart. Fix playbooks will come later,
in their own project, and only with a human approving each run.

**Why no sudo?**
The SSH key Semaphore uses reaches every server. Without sudo, even a bad playbook or a
stolen key can't change anything as root.

**Why does every playbook say `hosts: all` if it runs on one server?**
So the same playbook works on any server. Semaphore's Limit picks the one server, and the
first task refuses to continue if Limit was ever left out.

**Can I test a playbook myself?**
Test each command by hand over SSH as a normal user. Running the whole playbook happens from
Semaphore, by an admin, on a test server, because only Semaphore holds the key.

**Which servers can it run on?**
Only servers a Semaphore admin has prepared (a user with our public key) and added to the
inventory. IQ Core skips alerts for any other server.

**What if the server is down?**
Semaphore can't connect, and the ticket gets a note saying so.

## Glossary

| Term | Meaning |
|---|---|
| Playbook | A YAML file listing the steps to run on a server |
| Task | One step in a playbook |
| Module | What a task does: `command` runs a program, `debug` prints, `assert` checks a condition |
| `register` | Saves a task's output under a name for later tasks |
| Inventory | The list of servers Ansible may connect to |
| Limit | Narrows a run to some servers from the inventory; IQ Core always picks exactly one |
| Agentless | Nothing installed on target servers; Ansible connects over SSH |
| Template (Semaphore) | A playbook plus its settings, ready to run |
| Harness | The program around an AI model: loop, tools, permissions, triggers, record |
| Act tool | A tool that changes or runs something, as opposed to only reading |
