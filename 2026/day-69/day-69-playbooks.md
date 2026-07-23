# Day 69 — Ansible Playbooks

## 1. First Playbook
Installed Nginx, started/enabled it, added a custom index page (`install-nginx.yml`).
Ran twice → 1st run = `changed`, 2nd run = `ok`. This is **idempotency**: Ansible only changes what's actually different.

<img width="498" height="297" alt="image" src="https://github.com/user-attachments/assets/e2ebeab3-ce2e-47f7-ad8b-b2c0c270887a" />


## 2. Playbook Structure
- **Play** = targets a group of hosts, has `hosts:`, `become:`, `tasks:`
- **Task** = one module call inside a play
- A playbook can have **multiple plays**
- `become: true` at play level = applies to all tasks; at task level = overrides just that task
- If a task fails, Ansible stops that **host** but keeps running on other hosts

<img width="928" height="697" alt="image" src="https://github.com/user-attachments/assets/9a205fdb-d6d6-40c2-b597-2b22b4712902" />


<img width="870" height="195" alt="image" src="https://github.com/user-attachments/assets/02304c9e-be9b-47e0-9293-165242a790d8" />


## 3. Essential Modules
- `apt` / `yum` — install packages
- `service` — start/enable services
- `copy` — copy files or content
- `file` — create dirs, set permissions
- `command` — run a command, no shell features (safer, default choice)
- `shell` — run a command with pipes/redirects/etc.
- `lineinfile` — add/edit one line in a file

<img width="860" height="111" alt="image" src="https://github.com/user-attachments/assets/a8b2d4fb-0e0b-40bc-8f02-17ba8f26b265" />


<img width="761" height="291" alt="image" src="https://github.com/user-attachments/assets/f6368012-b469-4c01-8855-68ae68284bc5" />


## 4. Handlers
Run **only** when notified by a task that reports `changed` — used to restart services without unnecessary restarts.
```yaml
tasks:
  - name: Deploy config
    copy: ...
    notify: Restart Nginx

handlers:
  - name: Restart Nginx
    service:
      name: nginx
      state: restarted
```
<img width="873" height="728" alt="image" src="https://github.com/user-attachments/assets/0aee0be3-1f8d-451e-8bb7-6d17e0788284" />


Handlers run at the **end** of the play.

**Real bug hit today:** custom `nginx.conf` had `user nginx;`, but on Ubuntu the nginx user is `www-data` → service failed with `getpwnam("nginx") failed`. Fixed by changing to `user www-data;`.
Also: use `apt` on Ubuntu, not `yum` — `yum` silently no-ops instead of erroring clearly.


<img width="1376" height="842" alt="image" src="https://github.com/user-attachments/assets/61c8a482-778d-4349-ad60-2e0b47ee845f" />


<img width="849" height="425" alt="image" src="https://github.com/user-attachments/assets/b9794172-22b8-44b0-a494-302ffb372533" />


**Debug commands:**
```bash
sudo nginx -t
sudo systemctl status nginx --no-pager
sudo journalctl -xeu nginx.service --no-pager | tail -40
```

## 5. Dry Run, Diff, Verbosity, Limit, List
```bash
ansible-playbook site.yml --check           # preview, no changes
ansible-playbook site.yml --check --diff    # preview + show file diffs (most important combo)
ansible-playbook site.yml -v / -vv / -vvv   # more debug detail (vvv = SSH/connection debug)
ansible-playbook site.yml --limit node1     # run on specific host(s) only
ansible-playbook site.yml --list-hosts      # show which hosts would run
ansible-playbook site.yml --list-tasks      # show which tasks would run
```
`--check --diff` = safest way to preview production changes with zero risk.

## 6. Multiple Plays, One File
One playbook can have separate plays for different host groups (`web`, `app`, `db`), each with its own tasks. Groups with no matching hosts just get skipped.

---

### Key takeaways
- Idempotency is the whole point of Ansible.
- Match module to OS: `apt` (Ubuntu) vs `yum` (RHEL/CentOS).
- Handlers = restart only when something actually changed.
- Always `--check --diff` before running against production.
- Debug with `nginx -t`, `journalctl`, `systemctl status` — don't guess.
