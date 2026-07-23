# Day 70 – Ansible Variables, Facts, Conditionals & Loops

## 1. Variables

Variables make playbooks reusable.

```yaml
vars:
  app_name: terraweek-app
  app_port: 8080
  app_dir: "/opt/{{ app_name }}"
```

Use variables with:

```yaml
{{ app_name }}
{{ app_port }}
```

Override variables from CLI:

```bash
ansible-playbook variables-demo.yml -e "app_name=my-app app_port=9090"
```

`-e` / `--extra-vars` has very high precedence and overrides normal variables.

---

## 2. group_vars and host_vars

Variables can be stored outside playbooks.

```text
group_vars/
├── all.yml
├── web.yml
└── db.yml

host_vars/
└── node1.yml
```

### group_vars/all.yml

Applies to all hosts:

```yaml
app_env: development
common_packages:
  - vim
  - htop
  - tree
```

### group_vars/web.yml

Applies to the `web` group:

```yaml
http_port: 80
max_connections: 1000
```

### host_vars/node1.yml

Applies only to `node1`:

```yaml
max_connections: 2000
custom_message: "This is the primary web server"
```

`host_vars` filenames must match the inventory hostname.

Example:

```text
Inventory: node1
File: host_vars/node1.yml
```

Simplified precedence:

```text
group_vars
    ↓
host_vars
    ↓
higher precedence variables
    ↓
-e / --extra-vars
```

---

## 3. Ansible Facts

Ansible automatically gathers system information.

Useful facts:

```yaml
ansible_facts["distribution"]
ansible_facts["distribution_version"]
ansible_facts["hostname"]
ansible_facts["memtotal_mb"]
ansible_facts["default_ipv4"]["address"]
```

View facts:

```bash
ansible node1 -m setup
```

Filter facts:

```bash
ansible node1 -m setup -a "filter=ansible_os_family"
```

Facts can be used in playbooks:

```yaml
- name: Show OS
  debug:
    msg: "{{ ansible_facts['distribution'] }}"
```

Disable fact gathering when not needed:

```yaml
gather_facts: false
```

---

## 4. Conditionals with when

`when` controls whether a task runs.

Web server:

```yaml
when: "'web' in group_names"
```

Amazon Linux:

```yaml
when: ansible_facts["distribution"] == "Amazon"
```

Ubuntu:

```yaml
when: ansible_facts["distribution"] == "Ubuntu"
```

Low memory:

```yaml
when: ansible_facts["memtotal_mb"] < 1024
```

Multiple conditions (AND):

```yaml
when:
  - "'web' in group_names"
  - ansible_facts["memtotal_mb"] >= 512
```

OR condition:

```yaml
when: "'web' in group_names or 'app' in group_names"
```

If the condition is false, Ansible shows:

```text
skipping
```

---

## 5. Loops

Loops repeat a task for multiple items.

Example:

```yaml
users:
  - name: deploy
  - name: monitor
  - name: appuser
    groups: users
```

Use:

```yaml
- name: Create users
  user:
    name: "{{ item.name }}"
    state: present
  loop: "{{ users }}"
```

For directories:

```yaml
loop:
  - /opt/app/logs
  - /opt/app/config
  - /opt/app/data
```

`loop` is the modern recommended syntax.

Older syntax:

```yaml
with_items:
```

Both use:

```yaml
{{ item }}
```

### OS differences

My servers:

```text
node1 → Ubuntu
node2 → Amazon Linux
```

Ubuntu commonly uses:

```text
apt
sudo
```

Amazon Linux commonly uses:

```text
dnf/yum
wheel
```

Therefore, use conditionals for OS-specific tasks.

Example:

```yaml
when: ansible_facts["os_family"] == "Debian"
```

or:

```yaml
when: ansible_facts["os_family"] == "RedHat"
```

I also learned that Amazon Linux may already have `curl-minimal`, so installing the full `curl` package can cause a package conflict.

---

## 6. register and debug

`register` stores the result of a task.

```yaml
- name: Check disk
  command: df -h /
  register: disk_result
  changed_when: false
```

Use the result:

```yaml
{{ disk_result.stdout }}
```

`debug` displays information:

```yaml
- name: Show result
  debug:
    msg: "{{ disk_result.stdout }}"
```

---

## 7. Server Health Report

I combined facts, variables, conditionals, `register`, and `debug` to create a server report.

Commands used:

```yaml
command: df -h /
register: disk_result
```

```yaml
command: free -m
register: memory_result
```

```yaml
register: services_result
```

The report includes:

```text
Server
OS
IP
RAM
Disk usage
Running services
Timestamp
```

Report files:

```text
/tmp/server-report-node1.txt
/tmp/server-report-node2.txt
```

Check a report:

```bash
cat /tmp/server-report-node1.txt
```

---

## Key Takeaways

| Concept | Purpose |
|---|---|
| Variables | Store reusable values |
| group_vars | Variables for groups |
| host_vars | Variables for individual hosts |
| Facts | Gather system information |
| `when` | Run tasks conditionally |
| `loop` | Repeat tasks |
| `register` | Store task results |
| `debug` | Display information |

### Overall Flow

```text
Variables
   ↓
Facts
   ↓
Conditionals
   ↓
Loops
   ↓
Register Results
   ↓
Debug / Reports
```

Today I learned how to build more dynamic and reusable Ansible playbooks that can work across different hosts and operating systems.
