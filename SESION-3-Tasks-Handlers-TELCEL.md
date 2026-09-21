# Sesión 3: Tasks, Handlers y Notify
**TELCEL | HPE A&PS | 3 Horas | Workshop Ansible Intermedio**

> Tasks atómicas idempotentes, handlers con notify/listen, changed_when, Hardening SSH + Logrotate

## 📋 Objetivo

- Entender task atómica idempotente
- Handlers con notify y listen - reinicio condicional solo si cambia config
- changed_when / failed_when
- Labs: Hardening SSH + Rotación logs

---

## 1. Tasks Atómicas

Cada task 1 sola cosa, idempotente.

```yaml
# MAL - No idempotente
- shell: yum install -y httpd

# BIEN
- name: Instalar httpd
  dnf:
    name: httpd
    state: present
```

---

## 2. Handlers y Notify

Handler solo corre al final, una vez aunque 10 tasks hagan notify. Solo si task reporta changed.

```yaml
- name: Configurar SSH Hardening
  hosts: all
  become: yes
  tasks:
    - name: Copiar sshd_config
      template:
        src: sshd_config.j2
        dest: /etc/ssh/sshd_config
        validate: /usr/sbin/sshd -t -f %s
      notify: restart sshd

  handlers:
    - name: restart sshd
      service:
        name: sshd
        state: restarted
      listen: restart sshd
```

---

## 3. changed_when / failed_when

```yaml
- name: Validar config SSH
  command: /usr/sbin/sshd -t
  register: sshd_valid
  changed_when: false
  failed_when: sshd_valid.rc != 0
```

---

## 4. LAB 3.1 - Hardening SSH (45 min)

**inventory/group_vars/all.yml:**
```yaml
sshd_port: 2222
sshd_permit_root: "no"
sshd_password_auth: "no"
```

**templates/sshd_config.j2:**
```jinja
Port {{ sshd_port }}
PermitRootLogin {{ sshd_permit_root }}
PasswordAuthentication {{ sshd_password_auth }}
AllowUsers ansible telcel-admin
Banner /etc/issue.net
```

**playbooks/session3-sshd-hardening.yml:**
```yaml
---
- name: LAB 3.1 - Hardening SSH TELCEL
  hosts: all
  become: yes
  vars:
    sshd_custom_banner: |
      Acceso restringido - {{ company }} - {{ inventory_hostname }}
      Datacenter: {{ ansible_local.telcel.general.datacenter | default('N/A') }}
  tasks:
    - name: Crear banner
      copy:
        content: "{{ sshd_custom_banner }}"
        dest: /etc/issue.net

    - name: Configurar sshd_config
      template:
        src: templates/sshd_config.j2
        dest: /etc/ssh/sshd_config
        validate: /usr/sbin/sshd -t -f %s
        backup: yes
      notify: restart sshd

  handlers:
    - name: restart sshd
      service:
        name: sshd
        state: restarted
```

```bash
ansible-playbook playbooks/session3-sshd-hardening.yml --check --diff --limit servera
ansible-playbook playbooks/session3-sshd-hardening.yml --diff
ansible all -m shell -a "sshd -T | grep -E 'port|permitroot|passwordauth' -i"
```

---

## 5. LAB 3.2 - Logrotate con Handler (45 min)

**templates/logrotate-telcel.j2:**
```jinja
{{ log_path | default('/var/log/telcel/*.log') }} {
    daily
    rotate {{ log_rotate | default(7) }}
    compress
    missingok
    create 0644 {{ app_user | default('root') }} {{ app_user | default('root') }}
}
```

**playbooks/session3-logrotate.yml:**
```yaml
---
- name: LAB 3.2 - Logrotate TELCEL
  hosts: all
  become: yes
  vars:
    log_path: /var/log/telcel/*.log
    log_rotate: 14
  tasks:
    - file:
        path: /var/log/telcel
        state: directory
    - template:
        src: templates/logrotate-telcel.j2
        dest: /etc/logrotate.d/telcel
      notify: reload rsyslog

  handlers:
    - name: reload rsyslog
      service:
        name: rsyslog
        state: reloaded
```

```bash
ansible-playbook playbooks/session3-logrotate.yml --diff
ansible all -a "logrotate -f /etc/logrotate.d/telcel && ls -lh /var/log/telcel/"
```

---

## 6. Validación Final Sesión 3

```bash
ansible web -a "sshd -T | grep port"
ansible all -a "cat /etc/logrotate.d/telcel"
ansible all -a "cat /etc/issue.net"
```

Próxima: Sesión 4 - Loops y Condicionales
