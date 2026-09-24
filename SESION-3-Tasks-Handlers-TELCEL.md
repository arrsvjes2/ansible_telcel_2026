# Sesión 3 - Tasks, Handlers y Notify | TELCEL | HPE A&PS
## 100% Offline - Sin Internet - Sin Init - Container Friendly

> **Lab:** podman/docker containers sin systemd, sin internet, sin dnf. Todo con `file/copy/template/command/debug`.

---

## Objetivo

Entender task atómica idempotente, handler con notify/listen, changed_when/failed_when, aplicando config app + web + logs con validación condicional solo cuando cambia config.

**Módulos permitidos:** file, copy, template, lineinfile, user, group, command, shell (solo cat/grep), debug, stat, set_fact  
**Prohibido:** dnf/yum/package (requiere internet), service/systemd/reboot (no hay systemd)

---

## 1. Tasks Atómicas - Sin Internet

```yaml
# MAL - requiere internet
- dnf: name=httpd state=present

# BIEN - offline
- file: path=/etc/telcel state=directory mode=0755
- template: src=telcel-app.conf.j2 dest=/etc/telcel/app.conf mode=0644
- copy: content="<h1>TELCEL {{ inventory_hostname }}</h1>" dest=/var/www/telcel/index.html
```

---

## 2. Handlers y Notify - Offline

```yaml
- hosts: web
  tasks:
    - template:
        src: telcel-app.conf.j2
        dest: /etc/telcel/app.conf
        validate: sh -c "grep -q 'app_port' %s"
      notify: reload app

  handlers:
    - command: cat /etc/telcel/app.conf
      changed_when: false
      listen: reload app
    - debug: msg="Config cambió en {{ inventory_hostname }}"
      listen: reload app
```
segunda version:
```yaml
- hosts: web
  tasks:
    - template:
        src: telcel-app.conf.j2
        dest: /tmp/app.conf
        validate: sh -c "grep -q 'app_port' %s"
      notify: reload app

    - name: print message
      debug:
        msg: "Hello Class"

    - name: print http_port
      debug:
        msg: "{{ http_port }}"

  handlers:
    - command: cat /tmp/app.conf
      changed_when: false
      listen: reload app
    - debug: msg="Config cambió en {{ inventory_hostname }}"
      listen: reload app
```

Reglas:
- `validate` con `grep/cat` (no requiere binario httpd)
- `notify` solo si `changed`
- Handler al final una vez aunque 10 tasks hagan notify
- `--check --diff` muestra cambios

---

## 3. changed_when y failed_when - Offline

```yaml
- command: cat /etc/telcel/app.conf
  register: app_valid
  changed_when: false
  failed_when: app_valid.rc != 0

- shell: grep -q "^app_port" /etc/telcel/app.conf
  register: app_port_check
  changed_when: false
  failed_when: app_port_check.rc != 0
```

---

## 4. LAB 3.1 - Config App Web TELCEL (45 min) - Offline

**group_vars/web.yml**
```yaml
app_port: 8080
app_admin: noc@telcel.com.mx
app_server_name: "{{ inventory_hostname }}.telcel.lab"
app_version: 1.2.3
company: TELCEL
app_env: lab
app_docroot: /var/www/telcel
```

**templates/telcel-app.conf.j2 - 3 líneas mínimo**
```jinja
app_host={{ inventory_hostname }}
app_port={{ app_port }}
app_env={{ app_env | default('lab') }}
```

**templates/telcel-httpd.conf.j2 - Simula httpd sin instalar httpd**
```jinja
Listen {{ app_port }}
ServerName {{ app_server_name }}
DocumentRoot {{ app_docroot }}

<VirtualHost *:{{ app_port }}>
    ServerAdmin {{ app_admin }}
    ServerName {{ app_server_name }}
    DocumentRoot {{ app_docroot }}
    ErrorLog /var/log/telcel/error.log
    CustomLog /var/log/telcel/access.log combined
    # Host: {{ inventory_hostname }} - {{ ansible_host }}
    # Grupos: {{ group_names | join(',') }}
</VirtualHost>
```

**playbooks/session3-app.yml**
```yaml
---
- name: LAB 3.1 - Config App TELCEL offline sin dnf sin systemd
  hosts: web
  become: yes
  gather_facts: yes
  tasks:
    - file: path={{ item }} state=directory mode=0755
      loop: [/etc/telcel, /var/www/telcel, /var/log/telcel, /etc/httpd/conf.d]

    - copy:
        content: |
          <html><body>
          <h1>TELCEL LAB {{ inventory_hostname }}</h1>
          <p>Host: {{ inventory_hostname }} - {{ ansible_facts['default_ipv4']['address'] | default(ansible_host) }}</p>
          <p>Grupos: {{ group_names | join(',') }}</p>
          <p>Version: {{ app_version }}</p>
          </body></html>
        dest: /var/www/telcel/index.html
        mode: '0644'

    - template:
        src: telcel-app.conf.j2
        dest: /etc/telcel/app.conf
        mode: '0644'
        backup: yes
        validate: sh -c "grep -q 'app_port' %s && cat %s"
      notify: reload app
      register: app_config_result

    - template:
        src: telcel-httpd.conf.j2
        dest: /etc/httpd/conf.d/telcel.conf
        mode: '0644'
        backup: yes
        validate: sh -c "grep -q 'Listen' %s && cat %s"
      notify: reload app
      register: web_config_result

    - shell: grep -q "^app_port" /etc/telcel/app.conf && echo "OK app" && grep -q "^Listen" /etc/httpd/conf.d/telcel.conf && echo "OK web"
      changed_when: false
      when: app_config_result.changed or web_config_result.changed

  handlers:
    - command: cat /etc/telcel/app.conf
      changed_when: false
      listen: reload app
    - command: cat /etc/httpd/conf.d/telcel.conf
      changed_when: false
      listen: reload app
    - debug: msg="Nueva config {{ inventory_hostname }}:{{ app_port }}"
      listen: reload app

# ansible-playbook playbooks/session3-app.yml --check --diff --limit servera
# ansible web -a "cat /etc/telcel/app.conf && cat /etc/httpd/conf.d/telcel.conf"
```

---

## 5. LAB 3.2 - Rotación de Logs con Handler (45 min) - 100% Offline

**templates/logrotate-telcel.j2 - SIN systemctl**
```jinja
{{ log_path | default('/var/log/telcel/*.log') }} {
    daily
    rotate {{ log_rotate | default(7) }}
    compress
    delaycompress
    missingok
    notifempty
    create 0644 {{ app_user | default('root') }} {{ app_user | default('root') }}
    postrotate
        echo "Logs rotados {{ inventory_hostname }}" > /tmp/logrotate-done
    endscript
}
```

**playbooks/session3-logrotate.yml - Offline sin systemctl**
```yaml
---
- name: LAB 3.2 - Logrotate TELCEL offline sin systemd
  hosts: all
  become: yes
  vars:
    log_path: /var/log/telcel/*.log
    log_rotate: 14
    app_user: ansible
  tasks:
    - file: path=/var/log/telcel state=directory mode=0755 owner={{ app_user }}

    - copy:
        content: "Log {{ inventory_hostname }} {{ ansible_date_time.iso8601 | default('2025-09-15') }} Env {{ app_env | default('lab') }}"
        dest: /var/log/telcel/app-{{ inventory_hostname }}.log
        mode: '0644'

    - template:
        src: logrotate-telcel.j2
        dest: /etc/logrotate.d/telcel
        mode: '0644'
        validate: sh -c "grep -q 'daily' %s && cat %s"
      notify: show logrotate
      register: logrotate_result

    - command: cat /etc/logrotate.d/telcel
      changed_when: false
      register: logrotate_debug

    - debug: var=logrotate_debug.stdout_lines

    - shell: cp /var/log/telcel/app-{{ inventory_hostname }}.log /var/log/telcel/app-{{ inventory_hostname }}.log.1 && echo "" > /var/log/telcel/app-{{ inventory_hostname }}.log && ls -lh /var/log/telcel/
      changed_when: false
      when: logrotate_result.changed

  handlers:
    - command: cat /etc/logrotate.d/telcel
      changed_when: false
      listen: show logrotate
    - shell: ls -lh /var/log/telcel/ && cat /tmp/logrotate-done 2>/dev/null || echo "No postrotate"
      changed_when: false
      listen: show logrotate

# ansible all -a "cat /etc/logrotate.d/telcel && ls -lh /var/log/telcel/"
```

**Conceptos clave LAB 3.2:**
- `template` + `validate` con `grep -q daily`
- `notify` solo si changed
- Handler muestra config
- Simulación rotación con `cp + truncate` sin binario logrotate
- Idempotencia: 2da corrida 0 changed

---

## 6. LAB 3.3 - 2 templates + 1 handler listen compartido (15 min) - Offline

**templates/telcel-app.conf.j2 - Simple 3 líneas**
```jinja
app_host={{ inventory_hostname }}
app_port={{ app_port | default(8080) }}
app_env={{ app_env | default('lab') }}
```

**templates/telcel-app-v2.conf.j2 - Con magic vars Sesión 2**
```jinja
[app]
host={{ inventory_hostname }}
port={{ app_port }}
env={{ app_env }}
version={{ app_version }}
datacenter={{ ansible_local.telcel.general.datacenter | default('CDMX-02') }}

[cluster]
nodes={{ groups['web'] | join(',') }}
primary_db={{ groups['db'][0] | default('serverd') }}
primary_db_ip={{ hostvars[groups['db'][0]]['ansible_host'] | default('10.10.1.14') if groups['db'] is defined else 'N/A' }}

[custom_facts]
telcel_env={{ ansible_local.telcel.general.telcel_env | default('dev') }}
telcel_role={{ ansible_local.telcel.general.telcel_role | default('unknown') }}
```

**playbooks/session3-app-config.yml**
```yaml
---
- name: LAB 3.3 - 2 configs con handler listen compartido offline
  hosts: web
  become: yes
  gather_facts: yes
  tasks:
    - file: path=/etc/telcel state=directory mode=0755

    - template:
        src: telcel-app.conf.j2
        dest: /etc/telcel/app.conf
        mode: '0644'
        validate: sh -c "grep -q 'app_port' %s"
      notify: restart app

    - template:
        src: telcel-app-v2.conf.j2
        dest: /etc/telcel/app-v2.conf
        mode: '0644'
        validate: sh -c "grep -q 'host=' %s"
      notify: restart app

  handlers:
    - shell: cat /etc/telcel/app.conf && echo "---" && cat /etc/telcel/app-v2.conf
      changed_when: false
      listen: restart app
    - debug: msg="App config cambió en {{ inventory_hostname }}, 2 templates -> 1 handler - Offline OK"
      listen: restart app

# Prueba idempotencia:
# ansible-playbook playbooks/session3-app-config.yml --diff
# ansible-playbook playbooks/session3-app-config.yml --diff  # 2da vez 0 changed, NO handler
# ansible-playbook playbooks/session3-app-config.yml --check --diff
```

**Qué enseña LAB 3.3:**
- 2 tasks `template` hacen `notify: restart app`
- `listen: restart app` agrupa handlers
- Aunque 2 templates cambien, handler corre UNA vez al final
- Si nada cambia, handler NO corre (idempotencia)

---

## 7. Tarea Sesión 3 - Reto Offline Sin Internet

Reto: Crear handler con listen 'restart telcel' que haga:
- En web: cat /etc/httpd/conf.d/telcel.conf + cat /etc/telcel/app.conf
- En db: cat /etc/telcel/app.conf
- Disparado por 2 templates distintos
- Validar idempotencia con --check

```yaml
handlers:
  - shell: cat /etc/httpd/conf.d/telcel.conf && cat /etc/telcel/app.conf
    changed_when: false
    listen: restart telcel
    when: "'web' in group_names"
  - command: cat /etc/telcel/app.conf
    changed_when: false
    listen: restart telcel
    when: "'db' in group_names"
```

Validación:
```bash
ansible-playbook playbooks/session3-reto.yml --diff
# 2da ejecución: changed=0 => NO handler
```

---

## BONUS - Imagen con httpd preinstalado (1 vez con internet)

```dockerfile
FROM rockylinux:9
RUN dnf install -y httpd logrotate && dnf clean all
```
```bash
podman build -t lab-telcel-httpd:9 -f Dockerfile .
# Usar en lab offline
```

---

**Autor:** HPE A&PS | **Cliente:** TELCEL | **Sesión 3** | Septiembre 2026 | Offline Container Friendly
