# Sesión 2: Variables Inteligentes y Facts
**TELCEL | HPE A&PS | 3 Horas | Workshop Ansible Intermedio**

> Precedencia all → group → host, group_vars/host_vars, facts y custom facts /etc/ansible/facts.d/

## 📋 Objetivo

- Entender precedencia de variables (all → group → host) y por qué nunca hardcodear
- Usar group_vars y host_vars para separar config web vs db
- Usar facts nativos (ansible_facts) y custom facts telcel_env, patch_level
- Validar con ansible-inventory --graph --vars

---

## 1. Precedencia de Variables

Orden menor a mayor (el último gana):

```text
1. group_vars/all.yml                -> globales
2. group_vars/web.yml o db.yml       -> por grupo
3. host_vars/servera.yml             -> host específico
4. vars en playbook
5. extra_vars -e en CLI              -> máxima prioridad
```

Ejemplo real TELCEL:

```yaml
group_vars/all: ntp_server: time.telcel.com
group_vars/web: http_port: 8080
group_vars/db: http_port: 5432
host_vars/servera: http_port: 8081  # excepción
```

---

## 2. Estructura Profesional

```text
inventory/
├── hosts
├── group_vars/
│   ├── all.yml
│   ├── web.yml
│   └── db.yml
└── host_vars/
    ├── servera.yml
    └── servere.yml
```

**inventory/hosts:**
```ini
[web]
servera ansible_host=10.10.1.11
serverb ansible_host=10.10.1.12
serverc ansible_host=10.10.1.13

[db]
serverd ansible_host=10.10.1.14
servere ansible_host=10.10.1.15

[lab:children]
web
db
```

---

## 3. group_vars

**group_vars/all.yml**
```yaml
---
ntp_server: mx.pool.ntp.org
telcel_environment: lab
company: TELCEL
admin_email: admin@telcel.com
```

**group_vars/web.yml**
```yaml
---
http_port: 8080
http_service: httpd
app_user: telcel_web
packages_web:
  - httpd
  - mod_ssl
```

**group_vars/db.yml**
```yaml
---
http_port: 5432
db_service: postgresql
db_user: telcel_db
db_version: "15"
```

**host_vars/servera.yml**
```yaml
---
http_port: 8081
telcel_role: frontend-primary
```

---

## 4. Validación

```bash
ansible-inventory --graph
ansible-inventory --host servera --yaml | grep http_port
# Debe salir 8081 - ganó host_vars

ansible-inventory --host serverb --yaml | grep http_port
# 8080 - de group_vars/web
```

---

## 5. Facts Nativos

```bash
ansible servera -m setup | head -100
```

```yaml
- name: Mostrar facts clave
  hosts: all
  tasks:
    - debug:
        msg: "Soy {{ inventory_hostname }} con {{ ansible_facts['distribution'] }} y {{ ansible_facts['memtotal_mb'] }}MB RAM"
```

Desactivar si no necesitas: `gather_facts: no`

---

## 6. Custom Facts - /etc/ansible/facts.d/telcel.fact

Los facts nativos no saben si es prod/lab. Creamos custom facts.

**En cada managed:**
```bash
sudo mkdir -p /etc/ansible/facts.d/
sudo tee /etc/ansible/facts.d/telcel.fact <<'EOF'
[general]
telcel_env = prod
telcel_role = web
patch_level = 2025.09.15
datacenter = CDMX-01
owner = noc-telcel
EOF
```

Lectura:
```bash
ansible all -m setup -a "filter=ansible_local"
# ansible_local.telcel.general.telcel_env
ansible all -m debug -a "var=ansible_local.telcel.general.telcel_env"
```

---

## 7. Playbook con Variables y Facts

```yaml
---
- name: Sesion 2 - Variables y Facts TELCEL
  hosts: all
  gather_facts: yes
  tasks:
    - debug:
        msg: "Host {{ inventory_hostname }} -> grupo {{ group_names }} puerto {{ http_port }} env {{ telcel_environment }}"

    - debug:
        msg: "Custom fact env={{ ansible_local.telcel.general.telcel_env | default('NO DEFINIDO') }} patch={{ ansible_local.telcel.general.patch_level | default('N/A') }}"

    - name: Crear config con variables por grupo
      copy:
        content: |
          hostname={{ inventory_hostname }}
          port={{ http_port }}
          environment={{ telcel_environment }}
          datacenter={{ ansible_local.telcel.general.datacenter | default('desconocido') }}
        dest: /tmp/telcel-app.conf
```

Ejecución segura:
```bash
ansible-playbook playbooks/session2-variables.yml --check --diff
ansible-playbook playbooks/session2-variables.yml --diff
```

---

## 8. Labs Sesión 2

### LAB 2.1 - Preparar group_vars/host_vars (20 min)
```bash
mkdir -p inventory/group_vars inventory/host_vars
# Crear all.yml, web.yml, db.yml, host_vars/servera.yml
ansible-inventory --graph
ansible-inventory --host servera --yaml
```

### LAB 2.2 - Crear custom facts (30 min)
```yaml
# playbooks/session2-setup-facts.yml
- name: Crear custom facts TELCEL
  hosts: all
  become: yes
  tasks:
    - file:
        path: /etc/ansible/facts.d
        state: directory
    - copy:
        content: |
          [general]
          telcel_env = prod
          patch_level = 2025.09.15
        dest: /etc/ansible/facts.d/telcel.fact
```

```bash
ansible-playbook playbooks/session2-setup-facts.yml --diff
ansible all -m setup -a "filter=ansible_local"
```

### LAB 2.3 - Config por grupo (40 min)
```yaml
# playbooks/session2-app-config.yml
- name: LAB 2.3 - Configuracion por grupo
  hosts: all
  become: yes
  tasks:
    - copy:
        content: |
          server_port={{ http_port }}
          env={{ ansible_local.telcel.general.telcel_env }}
        dest: /etc/telcel-app.conf
```

---

## 9. Cheat Sheet

```bash
ansible-inventory --graph
ansible-inventory --host servera --yaml
ansible all -m setup | grep ansible_facts
ansible all -m setup -a filter=ansible_local
ansible all -m debug -a var=ansible_local.telcel.general.telcel_env
cat /etc/ansible/facts.d/telcel.fact
```

Próxima: Sesión 3 - Tasks, Handlers y Notify
