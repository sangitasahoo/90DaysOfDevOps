# Day 68 — Ansible Intro (Short Notes)

## What is Configuration Management?
Using code/automation to set up and maintain servers consistently, instead of doing it manually. Needed for consistency, speed, fewer errors, and easy scaling.

## Ansible vs Chef/Puppet/Salt
- **Ansible**: agentless, uses YAML, connects via SSH. Easiest to start with.
- **Chef/Puppet**: need an agent installed on each server.
- **Salt**: agent optional, very fast, uses ZeroMQ or SSH.

## Agentless
No software needs to be installed on the servers being managed. Ansible connects over **SSH** (Linux) or **WinRM** (Windows), runs the task, then leaves nothing behind.

## Architecture
- **Control Node** – where Ansible is installed (your machine).
- **Managed Nodes** – servers being configured (e.g. EC2 instances).
- **Inventory** – list of managed nodes.
- **Modules** – actual units of work (install package, copy file, etc.).
- **Playbooks** – YAML files defining what to run, on which hosts.

**Flow:** Control node → reads Inventory → runs Playbook → executes Modules → over SSH → on Managed Nodes.

## Where to Install Ansible
Only on the **control node** (e.g. your Ubuntu machine). Managed nodes just need SSH + Python — no agent required.

```bash
ansible --version
ansible <host> -m ping
```

## Ad-hoc Commands
Quick one-off tasks without a playbook.

```bash
ansible all -i inventory.ini -m command -a "uptime"
ansible web -i inventory.ini -m command -a "free -h"
ansible all -i inventory.ini -m command -a "df -h"
ansible web -i inventory.ini -m apt -a "name=git state=present" --become
ansible all -i inventory.ini -m copy -a "src=hello.txt dest=/tmp/hello.txt"
ansible web -i inventory.ini -m service -a "name=nginx state=restarted" --become
```

- `-m` = module, `-a` = arguments, `-i` = inventory file
- `--become` = run as sudo
- `command` = simple commands only; `shell` = supports pipes/redirects

## Inventory Groups & Patterns

```ini
[application:children]
web
app

[all_servers:children]
application
db
```

```bash
ansible application -i inventory.ini -m ping   # web + app
ansible 'web:app' -i inventory.ini -m ping     # OR
ansible 'all:!db' -i inventory.ini -m ping     # all except db
```

## ansible.cfg (skip typing -i every time)

```ini
[defaults]
inventory = inventory.ini
host_key_checking = False
remote_user = ubuntu
private_key_file = ~/your-key.pem
```

Then just run:
```bash
ansible all -m ping
```

Check it's working with `ansible --version` (look for `config file = ...`).

## Quick Tips
- SSH by default, no agent needed.
- Config file order: current folder → `~/.ansible.cfg` → `/etc/ansible/ansible.cfg`.
- `host_key_checking = False` is fine for practice, not for production.
- Ad-hoc = quick tasks; Playbooks = repeatable automation.
