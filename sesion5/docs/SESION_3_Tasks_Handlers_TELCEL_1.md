# Workshop Ansible Intermedio - Sesión 3
## Tasks, Handlers y Notify | 3 Horas | TELCEL | HPE A&PS
### Container Friendly con httpd - Sin systemd / Sin init

**Objetivo:** Entender task atómica idempotente, handler con notify y listen, changed_when/failed_when, aplicando config httpd con template + validate + handler sin necesidad de systemd.

---

## 1. Tasks Atómicas

```yaml
# MAL - No idempotente
- name: Instalar httpd
  shell: yum install -y httpd

# BIEN - Idempotente
- name: Instalar httpd
  dnf:
    name: httpd
    state: present
```

---

## 2. Handlers y Notify - El corazón (httpd sin init)

```yaml
- name: Configurar httpd TELCEL (Container Friendly - sin systemd)
  hosts: web
  become: yes
  tasks:
    - name: Copiar httpd config desde template
      template:
        src: telcel-httpd.conf.j2
        dest: /etc/httpd/conf.d/telcel.conf
        mode: '0644'
        validate: /usr/sbin/httpd -t -f /etc/httpd/conf/httpd.conf
      notify: reload httpd

  handlers:
    - name: reload httpd
      command: /usr/sbin/httpd -t
      changed_when: false
      listen: reload httpd

    - name: Mostrar config aplicada
      debug:
        msg: "Config httpd cambió en {{ inventory_hostname }}, validación OK"
      listen: reload httpd

# Reglas:
# - template + validate evita romper httpd
# - notify solo si changed
# - Handler corre al final una vez aunque 10 tasks hagan notify
# - En containers sin init NO usamos service: name=httpd, usamos command: httpd -t + debug
```

> **NOTA TELCEL:** Versión original usaba `sshd_config` + `service: sshd`. Tus nodos son containers sin init, no hay systemd. Cambiamos a `httpd conf.d` con `httpd -t` que funciona 100% en podman/docker.

---

## 3. LAB 3.1 - Config httpd TELCEL (45 min) - Container Friendly

**inventory/group_vars/web.yml**
```yaml
httpd_port: 8080
httpd_server_admin: noc@telcel.com.mx
httpd_server_name: "{{ inventory_hostname }}.telcel.lab"
app_version: 1.2.3
company: TELCEL
```

**templates/telcel-httpd.conf.j2 - Simple 3 líneas (mínimo)**
```jinja
Listen {{ httpd_port }}
ServerName {{ inventory_hostname }}.telcel.lab
DocumentRoot /var/www/telcel
```

**templates/telcel-httpd.conf.j2 - Completo TELCEL (recomendado para clase)**
```jinja
Listen {{ httpd_port }}
ServerName {{ httpd_server_name }}
DocumentRoot /var/www/telcel

<VirtualHost *:{{ httpd_port }}>
    ServerAdmin {{ httpd_server_admin }}
    ServerName {{ httpd_server_name }}
    DocumentRoot /var/www/telcel
    ErrorLog /var/log/httpd/telcel-error.log
    CustomLog /var/log/httpd/telcel-access.log combined
    # Host: {{ inventory_hostname }} - IP: {{ ansible_facts['default_ipv4']['address'] | default(ansible_host) }}
    # Grupos: {{ group_names | join(',') }}
</VirtualHost>
```

**playbooks/session3-httpd.yml**
```yaml
---
- name: LAB 3.1 - Config httpd TELCEL con template + handler (sin systemd)
  hosts: web
  become: yes
  gather_facts: yes

  tasks:
    - name: Crear directorios
      file:
        path: "{{ item }}"
        state: directory
        mode: '0755'
      loop:
        - /var/www/telcel
        - /var/log/httpd
        - /etc/httpd/conf.d

    - name: Crear index.html de prueba
      copy:
        content: |
          <html><body><h1>TELCEL LAB {{ inventory_hostname }}</h1>
          <p>Host: {{ inventory_hostname }} - {{ ansible_facts['default_ipv4']['address'] | default(ansible_host) }}</p>
          <p>Grupos: {{ group_names | join(',') }}</p>
          </body></html>
        dest: /var/www/telcel/index.html
        mode: '0644'

    - name: Configurar httpd TELCEL desde template
      template:
        src: templates/telcel-httpd.conf.j2
        dest: /etc/httpd/conf.d/telcel.conf
        mode: '0644'
        validate: /usr/sbin/httpd -t -f /etc/httpd/conf/httpd.conf
        backup: yes
      notify: reload httpd
      register: httpd_config_result

    - name: Validar sintaxis httpd
      command: /usr/sbin/httpd -t
      changed_when: false
      register: httpd_test
      when: httpd_config_result.changed

    - name: Debug si cambia
      debug:
        msg: "httpd config cambió en {{ inventory_hostname }}, handler reload httpd. RC={{ httpd_test.rc | default(0) }}"
      when: httpd_config_result.changed

  handlers:
    - name: reload httpd
      command: /usr/sbin/httpd -t
      changed_when: false
      listen: reload httpd

    - name: Mostrar nueva config
      debug:
        msg: "Nueva config httpd aplicada en {{ inventory_hostname }}:{{ httpd_port }}"
      listen: reload httpd

# Ejecución 100% compatible containers sin init
# ansible-playbook playbooks/session3-httpd.yml --check --diff --limit servera
# ansible-playbook playbooks/session3-httpd.yml --diff
# ansible web -a "cat /etc/httpd/conf.d/telcel.conf && httpd -t"
```

**Conceptos que se mantienen igual que sshd:**
- template + validate + backup
- notify solo si changed
- handler al final una sola vez
- --check --diff
- Idempotencia: segunda corrida = 0 changed

---

## 4. LAB 3.2 - Logrotate TELCEL con Handler

Ver archivo original - ya es container friendly (sin systemd).

---

## 5. Tarea

Reto: Crear handler con listen 'restart app' que valide httpd en web y liste logs en db, disparado por 2 templates distintos.
