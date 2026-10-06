# Workshop Ansible Intermedio - Sesión 3
## Tasks, Handlers y Notify | 3 Horas | TELCEL | HPE A&PS
### Container Friendly - Compatible con Podman/Docker

**Objetivo:** Entender task atómica idempotente, handler con notify y listen, changed_when/failed_when, aplicando config app y MOTD con reinicio condicional solo cuando cambia config.

---

## 1. Tasks Atómicas

Cada task debe hacer 1 sola cosa y ser idempotente. Usa módulos específicos, no shell/command salvo último recurso.

```yaml
# MAL - No idempotente
- name: Instalar httpd
  shell: yum install -y httpd

# BIEN - Idempotente
- name: Instalar httpd
  dnf:
    name: httpd
    state: present

# Módulos clave Sesión 3:
# package/dnf, copy, template, file, lineinfile, command, debug
```

---

## 2. Handlers y Notify - El corazón

```yaml
- name: Configurar MOTD TELCEL (Container Friendly)
  hosts: all
  become: yes
  tasks:
    - name: Copiar MOTD desde template
      template:
        src: telcel-motd.j2
        dest: /etc/motd.d/telcel
        mode: '0644'
      notify: refresh motd  # Solo dispara si changed

  handlers:
    - name: refresh motd
      command: cat /etc/motd.d/telcel
      changed_when: false
      listen: refresh motd  # Permite múltiples notify al mismo handler

# Reglas:
# - Handler solo corre al final del play, una vez aunque 10 tasks hagan notify
# - Si task no reporta changed, NO dispara handler
# - Con --check, Ansible dice si haría notify
# - En containers NO hay systemd/sshd, por eso usamos command/debug como handler demo
```

> **NOTA TELCEL:** En la versión original usábamos `sshd_config` + `service: sshd`. Como tus nodos son containers, no hay systemd ni sshd. Cambiamos a MOTD + config app, que enseña exactamente los mismos conceptos (template, validate, notify, handler) pero funciona 100% en podman/docker.

---

## 3. changed_when y failed_when

```yaml
- name: Validar MOTD
  command: cat /etc/motd.d/telcel
  register: motd_valid
  changed_when: false  # Este check nunca debe marcar changed
  failed_when: motd_valid.rc != 0

- name: Verificar logrotate config
  shell: logrotate -d /etc/logrotate.d/telcel 2>&1 | grep -q "error" && exit 1 || exit 0
  register: logrotate_check
  changed_when: false
  failed_when: logrotate_check.rc != 0
```

---

## 4. LAB 3.1 - MOTD y App Config TELCEL (45 min) - Container Friendly

**inventory/group_vars/all.yml agregar:**
```yaml
app_banner_enabled: true
app_env: lab
app_version: 1.2.3
motd_file: /etc/motd.d/telcel
httpd_port: 8080
company: TELCEL
```

**templates/telcel-motd.j2**
```jinja
=========================================
TELCEL LAB - {{ inventory_hostname }}
=========================================
Environment : {{ app_env }}
Datacenter  : {{ ansible_local.telcel.general.datacenter | default('CDMX-02') }}
Rol         : {{ ansible_local.telcel.general.telcel_role | default(group_names | join(',')) }}
Version     : {{ app_version }}
Managed by  : Ansible - {{ ansible_date_time.date | default('2025-09-15') }}
IP          : {{ ansible_facts['default_ipv4']['address'] | default(ansible_host | default('10.10.1.11')) }}
Grupos      : {{ group_names | join(',') }}
Banner ON   : {{ app_banner_enabled }}
=========================================
Acceso restringido - Solo personal autorizado
Company: {{ company | default('TELCEL') }}
Host: {{ inventory_hostname }} / {{ inventory_hostname_short }}
=========================================
```

**playbooks/session3-motd-hardening.yml**
```yaml
---
- name: LAB 3.1 - Banner MOTD y Config App TELCEL con template + handler
  hosts: all
  become: yes
  gather_facts: yes

  vars:
    motd_custom_banner: |
      =========================================
      Acceso restringido - {{ company | default('TELCEL') }} - {{ inventory_hostname }}
      Datacenter: {{ ansible_local.telcel.general.datacenter | default('CDMX-02') }}
      Rol: {{ ansible_local.telcel.general.telcel_role | default(group_names | join(',')) }}
      Solo personal autorizado
      IP: {{ ansible_facts['default_ipv4']['address'] | default(ansible_host) }}
      =========================================

  tasks:
    - name: Crear directorio /etc/motd.d
      file:
        path: /etc/motd.d
        state: directory
        mode: '0755'

    - name: Crear banner /etc/issue.net (compatibilidad)
      copy:
        content: "{{ motd_custom_banner }}"
        dest: /etc/issue.net
        mode: '0644'

    - name: Configurar MOTD TELCEL desde template (reemplaza sshd_config para containers)
      template:
        src: templates/telcel-motd.j2
        dest: /etc/motd.d/telcel
        mode: '0644'
        owner: root
        group: root
        backup: yes
      notify: refresh motd
      register: motd_config_result

    - name: Configurar /etc/motd principal (symlink o copy)
      copy:
        src: /etc/motd.d/telcel
        dest: /etc/motd
        remote_src: yes
        mode: '0644'
      when: motd_config_result.changed

    - name: Debug si cambia config
      debug:
        msg: "MOTD cambió en {{ inventory_hostname }}, se disparará handler refresh motd. Changed={{ motd_config_result.changed }}"
      when: motd_config_result.changed

  handlers:
    - name: refresh motd
      command: cat /etc/motd.d/telcel
      changed_when: false
      listen: refresh motd

    - name: Mostrar nuevo MOTD
      debug:
        msg: "Nuevo MOTD aplicado en {{ inventory_hostname }}"
      listen: refresh motd

# Ejecución segura - 100% compatible con containers podman/docker
# ansible-playbook playbooks/session3-motd-hardening.yml --check --diff --limit servera
# ansible-playbook playbooks/session3-motd-hardening.yml --diff --limit servera
# ansible-playbook playbooks/session3-motd-hardening.yml --diff
# Verificar: ansible all -a "cat /etc/motd && cat /etc/motd.d/telcel"
```

**Conceptos que se mantienen igual que con sshd_config:**
- template + backup
- notify solo si changed
- handler corre al final una sola vez
- --check --diff muestra cambios sin aplicar
- Idempotencia: segunda corrida = 0 changed

---

## 5. LAB 3.2 - Rotación de Logs con Handler Condicional (45 min)

**templates/logrotate-telcel.j2**
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

**playbooks/session3-logrotate.yml**
```yaml
---
- name: LAB 3.2 - Logrotate TELCEL con handler
  hosts: all
  become: yes
  vars:
    log_path: /var/log/telcel/*.log
    log_rotate: 14
    app_service: rsyslog

  tasks:
    - name: Crear directorio /var/log/telcel
      file:
        path: /var/log/telcel
        state: directory
        mode: '0755'
        owner: "{{ app_user | default('ansible') }}"

    - name: Crear logs dummy para prueba
      copy:
        content: "Log de prueba {{ inventory_hostname }} {{ ansible_date_time.iso8601 | default('2025-09-15') }}"
        dest: "/var/log/telcel/app-{{ inventory_hostname }}.log"
        mode: '0644'

    - name: Instalar logrotate config
      template:
        src: templates/logrotate-telcel.j2
        dest: /etc/logrotate.d/telcel
        mode: '0644'
      notify: reload logs

    - name: Forzar validación logrotate
      command: cat /etc/logrotate.d/telcel
      changed_when: false
      register: logrotate_debug

    - name: Mostrar debug logrotate
      debug:
        var: logrotate_debug.stdout_lines

  handlers:
    - name: reload logs
      command: ls -lh /var/log/telcel/
      changed_when: false
      listen: reload logs

# Probar rotación: ansible all -a "cat /etc/logrotate.d/telcel && ls -lh /var/log/telcel/"
```

---

## 6. Tarea Sesión 3

**Reto Container Friendly:** Crear handler con listen 'restart app' que haga `cat /etc/motd.d/telcel` en web y `ls /var/log/telcel/` en db, disparado por 2 templates distintos. Validar que si ningún template cambia, no hay handler (--check).

```yaml
# Idea:
handlers:
  - name: restart app web
    debug:
      msg: "Config web cambió en {{ inventory_hostname }}, recargando app"
    listen: restart app
    when: "'web' in group_names"

  - name: restart app db
    debug:
      msg: "Config db cambió en {{ inventory_hostname }}, recargando db"
    listen: restart app
    when: "'db' in group_names"
```
