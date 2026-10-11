# Sesión 7 - Uso y Creación de Roles Ansible - Ejecución en Nodo de CONTROL
## TELCEL Lab - Rocky Linux 10.1 + Ansible 2.16.16

> **Objetivo:** Ejecutar playbook y rol directamente en el nodo de control (localhost) con `connection: local`

---

## 1. Estructura Final para Control

```
sesion7-roles-control/
├── ansible.cfg
├── inventory-control.ini
├── playbooks/
│   └── session7-control.yml
└── roles/
    └── telcel_app/
        ├── defaults/main.yml
        ├── vars/main.yml
        ├── handlers/main.yml
        ├── meta/main.yml
        ├── tasks/main.yml
        └── templates/
            ├── telcel-app.conf.j2
            ├── telcel-httpd.conf.j2
            └── logrotate-telcel.j2
```

---

## 2. ansible.cfg - Para Control Local

```ini
[defaults]
inventory = ./inventory-control.ini
roles_path = ./roles
host_key_checking = False
interpreter_python = auto_silent
stdout_callback = yaml

[privilege_escalation]
become = False

```

**Claves:**
- `inventory = ./inventory-control.ini` → usa inventory local
- `roles_path = ./roles` → busca roles ahí
- `interpreter_python = auto_silent` → apaga warning de Rocky 10.1 Python 3.12
- `become = False` → control container sin systemd

---

## 3. inventory-control.ini - Ejecución Local

```ini
[control]
localhost ansible_connection=local ansible_python_interpreter=/usr/bin/python3.12

[control:vars]
app_port=8080
app_env=lab
log_level=INFO

```

**Explicación Men:**
- `localhost` → mismo nodo de control
- `ansible_connection=local` → **NO usa SSH**, ejecuta directo
- `ansible_python_interpreter=/usr/bin/python3.12` → Python de Rocky 10.1, apaga warning

Sin esto te da: `[WARNING]: Host 'serverc' is using discovered Python...`

---

## 4. Rol telcel_app Adaptado para Control

### 4.1 roles/telcel_app/defaults/main.yml (baja prioridad)

```yaml
---
app_port: 8080
app_env: lab
httpd_port: "{{ app_port }}"
httpd_docroot: /var/www/telcel-app
log_level: INFO
db_host: localhost
db_port: 5432

```

### 4.2 roles/telcel_app/vars/main.yml (alta prioridad)

```yaml
---
telcel_config_dir: /etc/telcel
telcel_log_dir: /var/log/telcel
httpd_docroot: /tmp/telcel-app

```

**Nota:** Si tu control es container sin root, cambia a:
```yaml
telcel_config_dir: /tmp/telcel/etc
telcel_log_dir: /tmp/telcel/log
```

### 4.3 roles/telcel_app/handlers/main.yml

```yaml
---
- name: Mostrar config app
  ansible.builtin.debug:
    msg: "app.conf actualizado en {{ telcel_config_dir }}/app.conf"
  listen: Mostrar config app

- name: Mostrar config httpd
  ansible.builtin.debug:
    msg: "httpd conf actualizado en /tmp/telcel-test/telcel.conf"
  listen: Mostrar config httpd

```

**Adaptado:** En control no hay systemd, usamos `debug` en vez de `systemd: restarted`

### 4.4 roles/telcel_app/tasks/main.yml - Principal

```yaml
---
- name: Validar variables requeridas
  ansible.builtin.assert:
    that:
      - app_port is defined
      - app_port | int > 1024
  tags:
    - always

- name: Crear directorios TELCEL en control
  ansible.builtin.file:
    path: "{{ item }}"
    state: directory
    mode: '0755'
  loop:
    - "{{ telcel_config_dir }}"
    - "{{ telcel_log_dir }}"
    - "{{ httpd_docroot }}"
    - "/tmp/telcel-test"
  tags:
    - config
    - deploy

- name: Bloque deploy con templates en control
  tags:
    - deploy
  block:
    - name: Template app.conf en control
      ansible.builtin.template:
        src: telcel-app.conf.j2
        dest: "{{ telcel_config_dir }}/app.conf"
        mode: '0644'
        backup: true
        validate: sh -c "grep -q 'app_port' %s"
      notify: Mostrar config app
      tags:
        - config
        - deploy

    - name: Template httpd conf en control
      ansible.builtin.template:
        src: telcel-httpd.conf.j2
        dest: "/tmp/telcel-test/telcel.conf"
        mode: '0644'
        backup: true
        validate: sh -c "grep -q 'Listen' %s"
      notify: Mostrar config httpd
      tags:
        - config
        - deploy

    - name: Template logrotate en control
      ansible.builtin.template:
        src: logrotate-telcel.j2
        dest: "/tmp/telcel-test/logrotate-telcel"
        mode: '0644'
      tags:
        - logs
        - deploy

    - name: Validar config
      ansible.builtin.shell:
        cmd: grep -q app_port {{ telcel_config_dir }}/app.conf && echo "DEPLOY OK"
      register: deploy_valid
      changed_when: false
      failed_when: "'OK' not in deploy_valid.stdout"
      tags:
        - validate

  rescue:
    - name: Rollback app.conf en control
      ansible.builtin.shell:
        cmd: cp {{ telcel_config_dir }}/app.conf.bak {{ telcel_config_dir }}/app.conf 2>/dev/null || echo "app_port=8080" > {{ telcel_config_dir }}/app.conf
      changed_when: true

  always:
    - name: Mostrar config final en control
      ansible.builtin.shell:
        cmd: cat {{ telcel_config_dir }}/app.conf
      changed_when: false
      register: final_config

    - name: Print config
      ansible.builtin.debug:
        msg: "{{ final_config.stdout_lines }}"

```

**Cambios vs Sesión 6 original:**
- ❌ Quitamos `dnf:` (no hay repos en control container)
- ❌ Quitamos `systemd:` (no hay systemd)
- ✅ Solo `file`, `template`, `shell`, `debug` → sí jalan en control

---

## 5. Templates

### 5.1 telcel-app.conf.j2

```jinja2
# {{ ansible_managed }}
# TELCEL APP Config - {{ inventory_hostname }}
# Ambiente: {{ app_env | default('lab') }}
# Fecha: {{ ansible_date_time.iso8601 }}

[app]
app_port={{ app_port | default(8080) }}
app_env={{ app_env | default('lab') }}
app_name=telcel-app
app_version=1.0.0

[server]
bind_address=0.0.0.0
port={{ app_port | default(8080) }}
workers={{ ansible_processor_vcpus | default(2) }}
log_level={{ log_level | default('INFO') }}

[database]
# Ejemplo con hostvars
db_host={{ db_host | default('localhost') }}
db_port={{ db_port | default(5432) }}
db_name=telcel_{{ app_env | default('lab') }}

[logging]
log_file=/var/log/telcel/app.log
log_rotate=true

```

### 5.2 telcel-httpd.conf.j2

```jinja2
# {{ ansible_managed }}
# TELCEL HTTPD Config - {{ inventory_hostname }}
# Ambiente: {{ app_env | default('lab') }}
# Fecha: {{ ansible_date_time.iso8601 }}

# Validacion requerida: debe contener 'Listen'
Listen {{ httpd_port | default(app_port) | default(8080) }}

<VirtualHost *:{{ httpd_port | default(app_port) | default(8080) }}>
    ServerName {{ inventory_hostname }}
    ServerAlias {{ ansible_fqdn | default(inventory_hostname) }}
    
    # DocumentRoot para app TELCEL
    DocumentRoot /var/www/telcel-app

    # Logs
    ErrorLog /var/log/httpd/telcel-error.log
    CustomLog /var/log/httpd/telcel-access.log combined

    # Proxy a la app en app_port
    ProxyPreserveHost On
    ProxyPass / http://127.0.0.1:{{ app_port | default(8080) }}/
    ProxyPassReverse / http://127.0.0.1:{{ app_port | default(8080) }}/

    <Directory /var/www/telcel-app>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    # Configuracion por ambiente
{% if app_env == 'prod' %}
    # Produccion - Seguridad extra
    ServerTokens Prod
    ServerSignature Off
    TraceEnable Off
    
    # SSL
    SSLEngine on
    SSLCertificateFile /etc/pki/tls/certs/telcel.crt
    SSLCertificateKeyFile /etc/pki/tls/private/telcel.key
{% else %}
    # Lab - Debug habilitado
    ServerTokens Full
    LogLevel debug
{% endif %}

    # Headers TELCEL
    Header always set X-Frame-Options "SAMEORIGIN"
    Header always set X-Content-Type-Options "nosniff"
    Header set X-App-Env "{{ app_env | default('lab') }}"

</VirtualHost>

# Configuracion adicional por grupo
{% if 'web' in group_names %}
# Config especifica para grupo web
<IfModule mod_status.c>
    <Location /server-status>
        SetHandler server-status
        Require ip 10.0.0.0/8
    </Location>
</IfModule>
{% endif %}

```

### 5.3 logrotate-telcel.j2

```jinja2
# {{ ansible_managed }}
{{ telcel_log_dir }}/app.log {
    daily
    rotate 7
}

```

---

## 6. Playbook para Control - session7-control.yml

```yaml
---
- name: Sesion 7 - Rol en nodo de CONTROL (localhost)
  hosts: control
  connection: local
  gather_facts: true
  become: false

  pre_tasks:
    - name: Mostrar que corre en control
      ansible.builtin.debug:
        msg: "Corriendo en CONTROL {{ inventory_hostname }} - Python {{ ansible_python_version }}"
      tags:
        - always

  roles:
    - role: telcel_app
      tags:
        - deploy

  post_tasks:
    - name: Validar archivos generados en control
      ansible.builtin.stat:
        path: "{{ item }}"
      loop:
        - "{{ telcel_config_dir }}/app.conf"
        - "/tmp/telcel-test/telcel.conf"
      register: files_check
      tags:
        - validate

    - name: Mostrar resultado validacion
      ansible.builtin.debug:
        msg: "{{ item.stat.path }} existe={{ item.stat.exists }} size={{ item.stat.size }}"
      loop: "{{ files_check.results }}"
      tags:
        - validate

```

**Claves Men:**
- `hosts: control` → grupo del inventory-control.ini
- `connection: local` → **ejecuta en el mismo nodo, sin SSH a servera/b/c**
- `become: false` → si tu control es container sin privilegios
- `roles: - telcel_app` → llama al rol

---

## 7. Comandos - Ejecución Directa en Control

### 7.1 Preparación en Rocky 10.1 Host

```bash
# Copia tar.gz a control
# Si estás en host Rocky 10.1:
podman cp sesion7-roles-control.tar.gz ansible_control_1:/root/

# Dentro del control
podman exec -it ansible_control_1 bash
cd /root
tar -xzf sesion7-roles-control.tar.gz
cd sesion7-roles-control
ls -R
```

### 7.2 Validación

```bash
# Verifica inventory local
cat inventory-control.ini
ansible-inventory --list
ansible-inventory --graph

# Ping a localhost (sin SSH)
ansible control -m ping
# Esperado: localhost | SUCCESS => pong

# Syntax check
ansible-playbook playbooks/session7-control.yml --syntax-check

# Lint con container (no toca 2.16.16)
podman run --rm -v $(pwd):/work:z -w /work ghcr.io/ansible/ansible-lint:latest
# O si estás dentro de control con podman vfs:
podman run --rm -v $(pwd):/work:z -w /work ghcr.io/ansible/ansible-lint:latest playbooks/session7-control.yml
```

### 7.3 Ejecución

```bash
# Dry-run (check mode)
ansible-playbook playbooks/session7-control.yml --diff --check

# Ejecución real en CONTROL
ansible-playbook playbooks/session7-control.yml --diff

# Con tags
ansible-playbook playbooks/session7-control.yml --tags config,deploy --diff
ansible-playbook playbooks/session7-control.yml --tags validate --diff

# Con vars prod
ansible-playbook playbooks/session7-control.yml -e "app_port=9090 app_env=prod" --diff

# Verbose
ansible-playbook playbooks/session7-control.yml --diff -v
```

### 7.4 Verificación Archivos Generados EN CONTROL

```bash
cat /etc/telcel/app.conf
# app_port=8080
# app_env=lab

cat /tmp/telcel-test/telcel.conf
# Listen 8080
# <VirtualHost *:8080>

cat /tmp/telcel-test/logrotate-telcel

ls -lh /etc/telcel/
ls -lh /tmp/telcel-test/
```

---

## 8. Diferencias: Ejecución Normal vs Control Local

| Concepto | Normal (servera/b/c) | Control Local (localhost) |
|----------|---------------------|---------------------------|
| inventory | `servera ansible_host=10.0.0.11` | `localhost ansible_connection=local` |
| playbook hosts | `hosts: web` | `hosts: control` |
| connection | `ssh` (default) | `local` |
| become | `true` (systemd/dnf) | `false` (container) |
| handlers | `systemd: restarted` | `debug:` |
| templates dest | `/etc/httpd/conf.d/` | `/tmp/telcel-test/` |

---

## 9. Troubleshooting Control

**Error: `Failed to find required executable systemd`**
→ Estás usando rol original con systemd en control. Usa el rol adaptado de este doc (con debug).

**Error: `Permission denied /etc/telcel`**
→ Tu control es container sin root. Cambia `vars/main.yml`:
```yaml
telcel_config_dir: /tmp/telcel/etc
```

**Warning Python interpreter**
→ Ya fix con `inventory-control.ini`:
```ini
ansible_python_interpreter=/usr/bin/python3.12
```

**Error podman dentro de control**
→ Control necesita `--privileged` y driver `vfs`:
```bash
echo -e "[storage]\ndriver=\"vfs\"" > /etc/containers/storage.conf
```

---

## 10. Checklist Final Sesión 7 Control

- [ ] `tar -xzf sesion7-roles-control.tar.gz`
- [ ] `cat inventory-control.ini` → tiene `ansible_connection=local`
- [ ] `ansible control -m ping` → pong
- [ ] `ansible-playbook playbooks/session7-control.yml --syntax-check` → OK
- [ ] `ansible-playbook playbooks/session7-control.yml --diff --check` → OK
- [ ] `ansible-playbook playbooks/session7-control.yml --diff` → 3 templates creados
- [ ] `cat /etc/telcel/app.conf` → contiene `app_port=8080`
- [ ] `podman run ... ansible-lint` → 0 errores

---

> **Fin Sesión 7 - TELCEL Lab - Ejecución en Control**
> Rocky 10.1 - Ansible 2.16.16 intacto - ansible-lint via container
> Tar: sesion7-roles-control.tar.gz
