# Day 69 — Ansible Playbooks

## Task 1: First Playbook (`install-nginx.yml`)

Installs Nginx, starts/enables it, and drops a custom index page.

```yaml
---
- name: Install and start Nginx on web servers
  hosts: web
  become: true

  tasks:
    - name: Install Nginx
      apt:
        name: nginx
        state: present

    - name: Start and enable Nginx
      service:
        name: nginx
        state: started
        enabled: true

    - name: Create a custom index page
      copy:
        content: "<h1>Deployed by Ansible - TerraWeek Server</h1>"
        dest: /usr/share/nginx/html/index.html
```

Run: `ansible-playbook -i inventory.ini install-nginx.yml`

**Idempotency**: run it once → tasks show `changed`. Run it again → tasks show `ok`. Ansible only makes changes when the actual state differs from the desired state.

Verify with: `curl http://<server-ip>`

---

## Task 2: Playbook Structure

```yaml
---                                    # YAML doc start
- name: Play name                      # PLAY — targets a group of hosts
  hosts: web                           # inventory group to run on
  become: true                         # run as root (sudo)

  tasks:                               # list of TASKS
    - name: Task name                  # TASK — one unit of work
      module_name:                     # MODULE — what Ansible does
        key: value                     # module arguments
```

**Q&A:**
1. **Play vs task** — a play maps a group of hosts to a set of actions; a task is one module call inside a play.
2. **Multiple plays** — yes, a playbook is a list of plays, each can target different hosts.
3. **`become: true`** — at play level, applies to every task by default; at task level, overrides just that one task.
4. **Task failure** — Ansible stops running further tasks *on that host* and marks it failed; other hosts keep going independently. Can override with `ignore_errors`, `failed_when`, or `block`/`rescue`.

---

## Task 3: Essential Modules (`essential-modules.yml`)

| Module | Purpose |
|---|---|
| `apt` / `yum` | install/remove packages |
| `service` | start/stop/enable services |
| `copy` | copy files or inline content to remote host |
| `file` | create dirs, set permissions/ownership |
| `command` | run a command, **no shell features** |
| `shell` | run a command **with shell features** (pipes, redirects) |
| `lineinfile` | add/modify a single line in a file |

```yaml
---
- name: Practice essential Ansible modules
  hosts: web
  become: true

  tasks:
    - name: Install multiple packages
      apt:
        name: [git, curl, wget, tree]
        state: present

    - name: Ensure Nginx is running
      service:
        name: nginx
        state: started
        enabled: true

    - name: Copy config file
      copy:
        src: files/app.conf
        dest: /etc/app.conf
        owner: root
        group: root
        mode: '0644'

    - name: Create application directory
      file:
        path: /opt/myapp
        state: directory
        owner: ec2-user
        mode: '0755'

    - name: Check disk space
      command: df -h
      register: disk_output

    - name: Print disk space
      debug:
        var: disk_output.stdout_lines

    - name: Count running processes
      shell: ps aux | wc -l
      register: process_count

    - name: Show process count
      debug:
        msg: "Total processes: {{ process_count.stdout }}"

    - name: Set timezone in environment
      lineinfile:
        path: /etc/environment
        line: 'TZ=Asia/Kolkata'
        create: true
```

**`command` vs `shell`**:
- `command` runs directly, no shell — no pipes/redirects/env vars/wildcards. Safer, no injection risk. **Default choice.**
- `shell` runs through `/bin/sh` — supports pipes, redirects, chaining. Use only when you actually need that syntax.

---

## Task 4: Handlers (`nginx-config.yml`)

Handlers only run when **notified** by a task that reports `changed`. Avoids unnecessary service restarts.

```yaml
---
- name: Configure Nginx with a custom config
  hosts: web
  become: true

  tasks:
    - name: Install Nginx
      apt:
        name: nginx
        state: present

    - name: Deploy Nginx config
      copy:
        src: files/nginx.conf
        dest: /etc/nginx/nginx.conf
        owner: root
        mode: '0644'
      notify: Restart Nginx

    - name: Deploy custom index page
      copy:
        content: "<h1>Managed by Ansible</h1><p>Server: {{ inventory_hostname }}</p>"
        dest: /usr/share/nginx/html/index.html

    - name: Ensure Nginx is running
      service:
        name: nginx
        state: started
        enabled: true

  handlers:
    - name: Restart Nginx
      service:
        name: nginx
        state: restarted
```

- First run: config file is new → `changed` → handler fires → Nginx restarts.
- Second run: nothing changed → `ok` → handler does **not** fire.
- Handlers run at the **end of the play**, not immediately when notified.

**Debugging note (real issue hit today):** on Ubuntu, the nginx worker user is `www-data`, not `nginx`. A custom `nginx.conf` with `user nginx;` will fail to start with `getpwnam("nginx") failed` since that system user doesn't exist. Fix: use `user www-data;` on Ubuntu. Also — always use `apt` not `yum` on Ubuntu hosts; `yum` tasks can misleadingly report `ok` without actually installing anything.

**Useful debug commands:**
```bash
sudo nginx -t                                   # test config syntax
sudo systemctl status nginx.service --no-pager
sudo journalctl -xeu nginx.service --no-pager | tail -40
sudo ss -tlnp | grep ':80 '                     # check for port conflicts
dpkg -l | grep nginx                            # confirm package install state
```

---

## Task 5: Dry Run, Diff, Verbosity, Limit, List

**Dry run (check mode)** — preview without changing anything:
```bash
ansible-playbook install-nginx.yml --check
```

**Diff mode** — show actual file differences:
```bash
ansible-playbook nginx-config.yml --check --diff
```

`--check --diff` together = the most important combo for production. It shows exactly what content would change, before it changes, with zero real side effects. Great for catching bad templates/vars before they overwrite a live config. Caveat: `command`/`shell` tasks can't meaningfully predict effects in check mode.

**Verbosity** — more output detail for debugging:
```bash
ansible-playbook install-nginx.yml -v     # module result data
ansible-playbook install-nginx.yml -vv    # + var/config loading detail
ansible-playbook install-nginx.yml -vvv   # + SSH/connection debugging
```

**Limit to specific hosts:**
```bash
ansible-playbook install-nginx.yml --limit web-server
ansible-playbook install-nginx.yml --limit 'node1,node2'
ansible-playbook install-nginx.yml --limit @install-nginx.retry   # retry only failed hosts
```

**List without running:**
```bash
ansible-playbook install-nginx.yml --list-hosts   # which hosts would be targeted
ansible-playbook install-nginx.yml --list-tasks   # which tasks would run, in order
```

Good pre-flight order before touching production:
```bash
--list-hosts → --list-tasks → --check --diff → (run for real)
```

---

## Task 6: Multiple Plays in One Playbook (`multi-play.yml`)

One playbook, three separate plays, each targeting a different inventory group.

```yaml
---
- name: Configure web servers
  hosts: web
  become: true
  tasks:
    - name: Install Nginx
      apt:
        name: nginx
        state: present
    - name: Start Nginx
      service:
        name: nginx
        state: started
        enabled: true

- name: Configure app servers
  hosts: app
  become: true
  tasks:
    - name: Install Node.js build dependencies
      apt:
        name: [gcc, make]
        state: present
    - name: Create app directory
      file:
        path: /opt/app
        state: directory
        mode: '0755'

- name: Configure database servers
  hosts: db
  become: true
  tasks:
    - name: Install MySQL client
      apt:
        name: mysql-client   # "mysql" on RHEL/CentOS, "mysql-client" on Ubuntu
        state: present
    - name: Create data directory
      file:
        path: /var/lib/appdata
        state: directory
        mode: '0700'
```

- If a group has no matching hosts in inventory, that play just skips cleanly — no error.
- To verify isolation: check that Nginx only lands on `[web]` hosts and `mysql-client` only lands on `[db]` hosts.

---

## Key takeaways from today

- **Idempotency** is the core promise of Ansible — running a playbook twice should only change things the second time if something actually drifted.
- **Ubuntu = `apt`, RHEL/CentOS = `yum`** — mixing these up gives misleading `ok` results instead of clear errors.
- **Handlers** exist to avoid unnecessary restarts — only fire on `notify` + an actual `changed` result.
- **Always dry-run production changes** with `--check --diff` first.
- **Debug with real tools**: `nginx -t`, `journalctl -xeu <service>`, `systemctl status`, `dpkg -l | grep <pkg>` — don't guess, check.
