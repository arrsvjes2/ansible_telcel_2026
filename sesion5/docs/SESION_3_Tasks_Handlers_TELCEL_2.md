# Sesión 3 - Tasks, Handlers y Notify | TELCEL | HPE A&PS
## 100% Container Friendly - Sin systemd / Sin init / Sin sshd

> Cliente TELCEL - Lab con podman/docker containers sin init - No usamos service/systemd/sshd

### Objetivo
Entender task atómica idempotente, handler con notify y listen, changed_when/failed_when, aplicando config httpd + app + logrotate con validación condicional solo cuando cambia config.

---

## 1. Tasks Atómicas

```yaml
# MAL - No idempotente
- shell: yum install -y httpd

# BIEN
- dnf: name=httpd state=present

# Módulos permitidos en containers sin init:
# dnf, copy, template, file, lineinfile, user, command, debug, shell
# PROHIBIDO en este lab: service, systemd, reboot (no hay systemd)
```

## 2. Handlers y Notify - httpd sin init

```yaml
- name: Configurar httpd TELCEL
  hosts: web
  tasks:
    - template:
        src: telcel-httpd.conf.j2
        dest: /etc/httpd/conf.d/telcel.conf
        validate: /usr/sbin/httpd -t -f /etc/httpd/conf/httpd.conf
      notify: reload httpd

  handlers:
    - command: /usr/sbin/httpd -t
      changed_when: false
      listen: reload httpd
    - debug:
        msg: "Config cambió en {{ inventory_hostname }}"
      listen: reload httpd
```

Reglas:
- template + validate evita romper httpd
- notify solo si changed
- Handler al final una vez aunque 10 tasks hagan notify
- Sin init: usamos command/debug, NO service

## 3. changed_when y failed_when

```yaml
- command: /usr/sbin/httpd -t
  register: httpd_valid
  changed_when: false
  failed_when: httpd_valid.rc != 0

- command: logrotate -d /etc/logrotate.d/telcel
  register: logrotate_check
  changed_when: false
  failed_when: "'error' in logrotate_check.stderr"
```

## 4. LAB 3.1 - httpd TELCEL (45 min)

**group_vars/web.yml**
```yaml
httpd_port: 8080
httpd_server_admin: noc@telcel.com.mx
httpd_server_name: "{{ inventory_hostname }}.telcel.lab"
app_version: 1.2.3
company: TELCEL
app_env: lab
```

**templates/telcel-httpd.conf.j2 - Simple 3 líneas**
```jinja
Listen {{ httpd_port }}
ServerName {{ inventory_hostname }}.telcel.lab
DocumentRoot /var/www/telcel
```

**templates/telcel-httpd.conf.j2 - Completo**
```jinja
Listen {{ httpd_port }}
ServerName {{ httpd_server_name }}
DocumentRoot /var/www/telcel
<VirtualHost *:{{ httpd_port }}>
    ServerAdmin {{ httpd_server_admin }}
    DocumentRoot /var/www/telcel
    ErrorLog /var/log/httpd/telcel-error.log
    CustomLog /var/log/httpd/telcel-access.log combined
    # Host: {{ inventory_hostname }} - {{ ansible_host }}
    # Grupos: {{ group_names | join(',') }}
</VirtualHost>
```

**playbooks/session3-httpd.yml**
```yaml
---
- name: LAB 3.1 - httpd TELCEL container-friendly
  hosts: web
  become: yes
  gather_facts: yes
  tasks:
    - file: path={{ item }} state=directory mode=0755
      loop: [/var/www/telcel, /var/log/httpd, /etc/httpd/conf.d]

    - copy:
        content: "<html><h1>TELCEL {{ inventory_hostname }}</h1><p>{{ group_names }}</p></html>"
        dest: /var/www/telcel/index.html
        mode: '0644'

    - template:
        src: templates/telcel-httpd.conf.j2
        dest: /etc/httpd/conf.d/telcel.conf
        mode: '0644'
        validate: /usr/sbin/httpd -t -f /etc/httpd/conf/httpd.conf
        backup: yes
      notify: reload httpd
      register: httpd_config_result

    - command: /usr/sbin/httpd -t
      changed_when: false
      when: httpd_config_result.changed

  handlers:
    - command: /usr/sbin/httpd -t
      changed_when: false
      listen: reload httpd
    - debug: msg="Nueva config {{ inventory_hostname }}:{{ httpd_port }}"
      listen: reload httpd

# ansible-playbook playbooks/session3-httpd.yml --check --diff --limit servera
# ansible web -a "cat /etc/httpd/conf.d/telcel.conf && httpd -t"
```

## 5. LAB 3.2 - Logrotate sin systemctl (45 min)

**templates/logrotate-telcel.j2 - SIN systemctl**
```jinja
{{ log_path | default('/var/log/telcel/*.log') }} {
    daily
    rotate {{ log_rotate | default(7) }}
    compress
    missingok
    notifempty
    create 0644 {{ app_user }} {{ app_user }}
    postrotate
        echo "Rotado {{ inventory_hostname }}" > /tmp/logrotate-done
    endscript
}
```

**playbooks/session3-logrotate.yml**
```yaml
---
- name: LAB 3.2 - Logrotate sin systemd
  hosts: all
  become: yes
  vars:
    log_path: /var/log/telcel/*.log
    log_rotate: 14
    app_user: ansible
  tasks:
    - file: path=/var/log/telcel state=directory mode=0755 owner={{ app_user }}
    - copy:
        content: "Log {{ inventory_hostname }} {{ ansible_date_time.iso8601 | default('now') }}"
        dest: /var/log/telcel/app-{{ inventory_hostname }}.log
        mode: '0644'

    - template:
        src: templates/logrotate-telcel.j2
        dest: /etc/logrotate.d/telcel
        mode: '0644'
      notify: show logrotate

    - command: logrotate -d /etc/logrotate.d/telcel
      changed_when: false
      register: logrotate_debug

    - debug: var=logrotate_debug.stderr_lines

  handlers:
    - command: cat /etc/logrotate.d/telcel
      changed_when: false
      listen: show logrotate
    - command: ls -lh /var/log/telcel/
      changed_when: false
      listen: show logrotate

# ansible all -a "cat /etc/logrotate.d/telcel && ls -lh /var/log/telcel/"
```

## 6. LAB 3.3 - 2 templates + 1 handler listen (15 min)

**templates/telcel-app.conf.j2 - Simple 3 líneas**
```jinja
app_host={{ inventory_hostname }}
app_port={{ httpd_port | default(8080) }}
app_env={{ app_env | default('lab') }}
```

**templates/telcel-app-v2.conf.j2**
```jinja
[app]
host={{ inventory_hostname }}
port={{ httpd_port }}
env={{ app_env }}
version={{ app_version }}
datacenter={{ ansible_local.telcel.general.datacenter | default('CDMX-02') }}

[cluster]
nodes={{ groups['web'] | join(',') }}
primary_db={{ groups['db'][0] | default('serverd') }}
primary_db_ip={{ hostvars[groups['db'][0]]['ansible_host'] | default('10.10.1.14') if groups['db'] is defined else 'N/A' }}
```

**playbooks/session3-app-config.yml**
```yaml
---
- name: LAB 3.3 - 2 configs con handler listen compartido
  hosts: web
  become: yes
  tasks:
    - template: src=telcel-app.conf.j2 dest=/etc/telcel/app.conf mode=0644
      notify: restart app
    - template: src=telcel-app-v2.conf.j2 dest=/etc/telcel/app-v2.conf mode=0644
      notify: restart app

  handlers:
    - command: cat /etc/telcel/app.conf && cat /etc/telcel/app-v2.conf
      changed_when: false
      listen: restart app
    - debug: msg="App config cambió en {{ inventory_hostname }}, 2 templates -> 1 handler"
      listen: restart app

# Segunda corrida debe ser 0 changed y NO disparar handler
# ansible-playbook playbooks/session3-app-config.yml --diff
# ansible-playbook playbooks/session3-app-config.yml --diff
```

## 7. Tarea Sesión 3

Reto container-friendly sin systemd:
- Handler listen 'restart telcel' que haga httpd -t en web y cat /etc/telcel/app.conf en db
- Disparado por 2 templates distintos
- Validar idempotencia con --check --diff
