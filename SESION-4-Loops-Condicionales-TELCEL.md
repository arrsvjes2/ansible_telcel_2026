# Sesión 4: Lógica Avanzada - Loops y Condicionales
**TELCEL | HPE A&PS | 3 Horas | Workshop Ansible Intermedio**

> loop, when, register, until/retries, dict2items, gestión masiva usuarios + auditoría paquetes

## 📋 Objetivo

- Dominar loop moderno, loop_control, loops con diccionarios
- when con facts, group_names, inventory_hostname, register
- until/retries para espera servicios
- Labs: User management masivo + Package audit

---

## 1. Loop Moderno

`loop` reemplaza `with_items` desde Ansible 2.5+

```yaml
- name: Crear usuarios
  user:
    name: "{{ item }}"
    state: present
  loop:
    - alice
    - bob

# Lista de diccionarios
- user:
    name: "{{ item.name }}"
    groups: "{{ item.groups }}"
  loop:
    - { name: 'noc1', groups: 'wheel,telcel' }
    - { name: 'dba1', groups: 'dba' }

# loop_control para logs limpios
- dnf:
    name: "{{ item }}"
  loop: "{{ packages_web }}"
  loop_control:
    label: "{{ item }}"
```

---

## 2. Loops con dict2items

```yaml
# group_vars/all.yml
telcel_users:
  web_team:
    - alice
    - bob
  db_team:
    - carol
    - dave

- debug:
    msg: "Equipo {{ item.key }} usuario {{ item.value }}"
  loop: "{{ telcel_users | dict2items }}"

- user:
    name: "{{ item.1 }}"
    groups: "{{ item.0.key }}"
  loop: "{{ telcel_users | dict2items | subelements('value') }}"
```

---

## 3. Condicionales when

```yaml
- debug:
    msg: "Soy web"
  when: "'web' in group_names"

- service:
    name: httpd
    state: started
  when: ansible_local.telcel.general.telcel_env == "prod"

- debug:
    msg: "Web prod"
  when:
    - "'web' in group_names"
    - ansible_local.telcel.general.telcel_env == "prod"

# when con register
- stat:
    path: /etc/telcel-app.conf
  register: app_conf

- copy:
    src: telcel-app.conf
    dest: /etc/telcel-app.conf
  when: not app_conf.stat.exists
```

---

## 4. register, until, retries

```yaml
- uri:
    url: "http://{{ ansible_facts['default_ipv4']['address'] }}:{{ http_port }}"
    status_code: 200
  register: result
  until: result.status == 200
  retries: 10
  delay: 5

- shell: df / --output=avail | tail -1
  register: disk_free
  changed_when: false
```

---

## 5. LAB 4.1 - Gestión Masiva Usuarios (60 min)

**group_vars/all.yml:**
```yaml
telcel_user_list:
  - { name: 'noc01', groups: 'wheel', shell: '/bin/bash', env: 'all' }
  - { name: 'noc02', groups: 'wheel', shell: '/bin/bash', env: 'all' }
  - { name: 'dba01', groups: 'dba', shell: '/bin/bash', env: 'db' }
  - { name: 'dba02', groups: 'dba', shell: '/bin/bash', env: 'db' }
  - { name: 'dev01', groups: 'telcel', shell: '/bin/bash', env: 'web' }
  - { name: 'dev02', groups: 'telcel', shell: '/bin/bash', env: 'web' }
  - { name: 'auditor', groups: '', shell: '/bin/false', env: 'all' }
  - { name: 'deploy', groups: 'wheel', shell: '/bin/bash', env: 'prod' }
```

**playbooks/session4-user-management.yml:**
```yaml
---
- name: LAB 4.1 - User Management Masivo
  hosts: all
  become: yes
  tasks:
    - group:
        name: "{{ item }}"
      loop: [wheel, dba, telcel]

    - user:
        name: "{{ item.name }}"
        groups: "{{ item.groups }}"
        shell: "{{ item.shell }}"
      loop: "{{ telcel_user_list }}"
      loop_control:
        label: "{{ item.name }}"
      when: item.env == 'all' or item.env in group_names or (item.env == 'prod' and ansible_local.telcel.general.telcel_env | default('lab') == 'prod')
```

```bash
ansible-playbook playbooks/session4-user-management.yml --check --diff --limit servera
ansible-playbook playbooks/session4-user-management.yml --diff
ansible all -a "cat /etc/passwd | grep -E 'noc|dba|dev|auditor'"
```

---

## 6. LAB 4.2 - Auditoría Paquetes (60 min)

**playbooks/session4-package-audit.yml:**
```yaml
---
- name: LAB 4.2 - Package Audit
  hosts: all
  become: yes
  vars:
    critical_packages: [vim, htop, tmux, git, rsyslog]
    forbidden_packages: [telnet, rsh]
  tasks:
    - shell: df / --output=avail -B1 | tail -1
      register: disk_avail
      changed_when: false

    - fail:
        msg: "No hay espacio suficiente en {{ inventory_hostname }}"
      when: disk_avail.stdout | int < 524288000

    - dnf:
        name: "{{ item }}"
        state: present
      loop: "{{ critical_packages }}"
      loop_control:
        label: "{{ item }}"
      register: install_result
      until: install_result is succeeded
      retries: 3

    - shell: rpm -q {{ item }} && echo "FOUND" || echo "NOTFOUND"
      loop: "{{ forbidden_packages }}"
      register: forbidden_check
      changed_when: false

    - dnf:
        name: "{{ item.item }}"
        state: absent
      loop: "{{ forbidden_check.results }}"
      when: "'FOUND' in item.stdout"

    - copy:
        content: |
          Reporte Audit - {{ inventory_hostname }} - {{ ansible_date_time.iso8601 }}
          Env: {{ ansible_local.telcel.general.telcel_env | default('N/A') }}
          Memoria: {{ ansible_facts['memtotal_mb'] }} MB
        dest: /tmp/package-audit.txt
```

```bash
ansible-playbook playbooks/session4-package-audit.yml --diff
ansible all -a "cat /tmp/package-audit.txt"
```

---

## 7. Cheat Sheet Sesión 3 y 4

```bash
# Handlers
notify: restart sshd + handlers: -> solo si changed

# Loops
loop: {{ lista }} + loop_control label: {{ item.name }}

# Condicionales
when: "'web' in group_names"
when: ansible_local.telcel.general.telcel_env == 'prod'

# Reintentos
register: result + until: result.rc == 0 + retries: 5

# Dict loops
dict2items | subelements('value')

# Seguro
--limit servera --check --diff
```

Próxima: Sesión 5 - Ejecución Controlada (tags, block/rescue/always)
