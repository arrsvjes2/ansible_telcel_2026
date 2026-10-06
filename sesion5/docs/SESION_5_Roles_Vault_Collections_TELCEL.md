
# Sesión 5 - Roles, Collections y Ansible Vault | TELCEL | HPE A&PS
## 100% Offline - Sin Internet - Sin Init

### Objetivo
Roles, galaxy offline, vault, site.yml orquestador. Todo sin internet.

---

## 1. Roles

```
roles/telcel_app/
  tasks/main.yml
  handlers/main.yml
  templates/
  defaults/main.yml
  vars/main.yml
```

`ansible-galaxy role init roles/telcel_app --offline`

---

## 2. Galaxy y Collections Offline

```ini
# ansible.cfg
[defaults]
roles_path = ./roles
collections_path = ./collections
inventory = inventory/hosts
host_key_checking = False
vault_password_file = ~/.vault_pass
```

---

## 3. Vault

```bash
echo "TelcelVault2025" > ~/.vault_pass && chmod 600 ~/.vault_pass
ansible-vault create --vault-password-file ~/.vault_pass inventory/group_vars/web/vault.yml
```

```yaml
# vault.yml cifrado
vault_db_password: "SuperSecretDB2025!"
vault_api_key: "TELCEL-API-XYZ-123"

# web.yml
db_password: "{{ vault_db_password }}"

# tasks
- template: src=app-secrets.conf.j2 dest=/etc/telcel/secrets.conf mode=0600
  no_log: true
```

---

## 4. LAB 5.1 - Role telcel_app (45 min)

**defaults/main.yml**
```yaml
app_port: 8080
app_env: lab
app_docroot: /var/www/telcel
app_admin: noc@telcel.com.mx
app_version: 1.2.3
company: TELCEL
```

**templates/telcel-app.conf.j2 - 3 líneas**
```jinja
app_host={{ inventory_hostname }}
app_port={{ app_port }}
app_env={{ app_env }}
```

**templates/telcel-httpd.conf.j2**
```jinja
Listen {{ app_port }}
ServerName {{ inventory_hostname }}.telcel.lab
DocumentRoot {{ app_docroot }}
<VirtualHost *:{{ app_port }}>
    ServerAdmin {{ app_admin }}
    DocumentRoot {{ app_docroot }}
</VirtualHost>
```

**tasks/main.yml - Offline sin dnf**
```yaml
---
- file: path={{ item }} state=directory mode=0755
  loop: [/etc/telcel, /var/www/telcel, /var/log/telcel, /etc/httpd/conf.d]

- copy:
    content: "<h1>TELCEL {{ inventory_hostname }}</h1>"
    dest: "{{ app_docroot }}/index.html"

- template:
    src: telcel-app.conf.j2
    dest: /etc/telcel/app.conf
    validate: sh -c "grep -q 'app_port' %s"
  notify: reload telcel_app

- template:
    src: telcel-httpd.conf.j2
    dest: /etc/httpd/conf.d/telcel.conf
    validate: sh -c "grep -q 'Listen' %s"
  notify: reload telcel_app
```

**handlers/main.yml**
```yaml
---
- command: cat /etc/telcel/app.conf
  changed_when: false
  listen: reload telcel_app
- debug: msg="Role telcel_app {{ inventory_hostname }}:{{ app_port }}"
  listen: reload telcel_app
```

---

## 5. LAB 5.2 - Role telcel_logrotate (30 min)

**defaults/main.yml**
```yaml
log_path: /var/log/telcel/*.log
log_rotate: 14
app_user: ansible
```

**templates/logrotate-telcel.j2**
```jinja
{{ log_path }} {
    daily
    rotate {{ log_rotate }}
    missingok
    notifempty
    create 0644 {{ app_user }} {{ app_user }}
    postrotate
        echo "Rotado {{ inventory_hostname }}" > /tmp/logrotate-done
    endscript
}
```

**tasks/main.yml**
```yaml
---
- file: path=/var/log/telcel state=directory owner={{ app_user }}
- copy:
    content: "Log {{ inventory_hostname }}"
    dest: /var/log/telcel/app-{{ inventory_hostname }}.log
- template:
    src: logrotate-telcel.j2
    dest: /etc/logrotate.d/telcel
    validate: sh -c "grep -q 'daily' %s"
  notify: show logrotate
```

---

## 6. LAB 5.3 - Vault (30 min)

```bash
ansible-vault create --vault-password-file ~/.vault_pass inventory/group_vars/web/vault.yml
# vault_db_password: "SuperSecretDB2025!"
ansible-playbook site.yml --vault-password-file ~/.vault_pass --diff
```

Template con vault + no_log + 0600

---

## 7. LAB 5.4 - site.yml

```yaml
---
- hosts: web
  become: yes
  roles: [telcel_app, telcel_logrotate]

- hosts: db
  become: yes
  roles: [telcel_logrotate]

- hosts: all
  become: yes
  tasks:
    - copy: content="TELCEL {{ inventory_hostname }}" dest=/etc/motd
```

```bash
ansible-playbook site.yml --vault-password-file ~/.vault_pass --check --diff
ansible-playbook site.yml --vault-password-file ~/.vault_pass --diff
# 2da vez 0 changed
```

---

## 8. Reto Final

Crear telcel_base + telcel_app(vault) + telcel_logrotate + site.yml offline.

---

**Entregable:** Roles, inventory con vault cifrado, site.yml, ansible.cfg
