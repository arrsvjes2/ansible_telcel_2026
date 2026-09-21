# Workshop Ansible Intermedio - Sesión 1 - Referencias Oficiales
### TELCEL | HPE Advisory & Professional Services | 3 Horas

> Fundamentos, Buenas Prácticas, Arquitectura, Estructura Profesional, Modo Seguro `--check --diff --limit`, Linting y Lab Podman

---

## 📚 Tabla de Contenidos

1. [Arquitectura Ansible](#1-arquitectura-ansible)
2. [Idempotencia](#2-idempotencia)
3. [Estructura Profesional de Proyecto](#3-estructura-profesional)
4. [ansible.cfg](#4-ansiblecfg)
5. [Inventario](#5-inventario)
6. [Modo Seguro --check --diff --limit](#6-modo-seguro)
7. [Linting y Validación](#7-linting)
8. [Lab Podman-Compose](#8-lab-podman)
9. [Playbook day0-setup.yml](#9-day0-setup)
10. [Cheat Sheet Sesión 1](#10-cheat-sheet)

---

## 1. Arquitectura Ansible

Ansible es **agentless** y funciona en modelo **Push** vía SSH. Solo necesitas Python en los nodos gestionados.

- **Control Node**: Donde ejecutas `ansible-playbook` (tu `control` con Rocky 9 / RHEL10)
- **Managed Nodes**: `servera-e` - solo SSH + Python 3.9+
- **Sin agentes**: No hay demonio, no hay base de datos central

**Referencia Oficial:**
- [Ansible Basic Concepts - How Ansible Works](https://docs.ansible.com/ansible/latest/getting_started/basic_concepts.html)
- [Ansible Architecture](https://docs.ansible.com/ansible/latest/reference_appendices/glossary.html)

```bash
ansible --version
ansible all -m ping
```

---

## 2. Idempotencia

> *Ansible's Resources follow a declarative approach, determining the necessary steps to achieve the final state and notifying you if any actions were taken to reach that state.*

**Regla HPE:** Usa módulos específicos idempotentes, evita `shell` / `command` salvo último recurso.

```yaml
# MAL - No idempotente
- shell: yum install -y httpd

# BIEN - Idempotente
- name: Instalar httpd
  ansible.builtin.dnf:
    name: httpd
    state: present
```

**Referencia:**
- [Ansible Idempotency Explained](https://docs.ansible.com/ansible/latest/reference_appendices/glossary.html#term-Idempotency)

---

## 3. Estructura Profesional

Esta es la estructura que usamos en el lab TELCEL:

```text
/home/ansible/playbooks/
├── ansible.cfg
├── inventory/
│   ├── hosts
│   └── group_vars/
│       ├── all.yml
│       ├── web.yml
│       └── db.yml
│   └── host_vars/
│       ├── servera.yml
│       └── servere.yml
├── playbooks/
│   ├── day0-setup.yml
│   └── session2-variables.yml
└── templates/
    ├── motd.j2
    └── sshd_config.j2
```

**Referencia Oficial - Best Practices:**
- [Ansible Best Practices - Directory Layout](https://docs.ansible.com/ansible/latest/tips_tricks/sample_setup.html)
- [Sample Setup - Alternative Directory Layout](https://docs.ansible.com/ansible/latest/tips_tricks/sample_setup.html)

> Tip: `ansible.cfg` en la raíz del proyecto tiene prioridad sobre `~/.ansible.cfg` y `/etc/ansible/ansible.cfg`

---

## 4. ansible.cfg

Configura comportamiento por defecto: inventario, usuario remoto, validación SSH, etc.

**Orden de búsqueda (precedence):**
1. `ANSIBLE_CONFIG` env var
2. `./ansible.cfg` (en directorio actual)
3. `~/.ansible.cfg`
4. `/etc/ansible/ansible.cfg`

**Ejemplo usado en Sesión 1 - TELCEL:**

```ini
[defaults]
inventory = ./inventory/hosts
remote_user = ansible
host_key_checking = False
retry_files_enabled = False
forks = 20
log_path = ./ansible.log
interpreter_python = auto_silent
collections_paths = ./collections:~/.ansible/collections:/usr/share/ansible/collections

[privilege_escalation]
become = False
become_method = sudo
```

**Validación:**
```bash
ansible-config dump --only-changed
ansible-config init --disabled | grep -A2 inventory
```

**Referencias:**
- [Ansible Configuration File](https://docs.ansible.com/ansible/latest/reference_appendices/config.html)
- [ansible.cfg Explained - Search Order, Sections](https://www.golinuxcloud.com/ansible-cfg-explained/)

---

## 5. Inventario

**Referencia Oficial:**
- [How to build your inventory](https://docs.ansible.com/ansible/latest/inventory_guide/intro_inventory.html)
- [Inventory Tips - Group by function, separate prod and staging](https://docs.ansible.com/ansible/latest/tips_tricks/tips_tricks.html#inventory-tips)

**Ejemplo TELCEL `inventory/hosts`:**

```ini
[web]
servera ansible_host=10.10.1.11 ansible_user=ansible
serverb ansible_host=10.10.1.12
serverc ansible_host=10.10.1.13

[db]
serverd ansible_host=10.10.1.14
servere ansible_host=10.10.1.15

[lab:children]
web
db

[lab:vars]
ansible_ssh_common_args='-o StrictHostKeyChecking=no'
```

**Comandos clave de validación:**

```bash
# Ver grafo
ansible-inventory --graph

# Ver variables efectivas de un host (precedencia all -> group -> host)
ansible-inventory --host servera --yaml

# Ping a todos
ansible all -m ping

# Solo web
ansible web -m ping
```

---

## 6. Modo Seguro --check --diff --limit

Tu mantra HPE para producción: **Nunca ejecutes sin check primero.**

**Referencia Oficial:**
- [Check Mode (Dry Run)](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_checkmode.html)
- [Diff Mode](https://docs.ansible.com/ansible/latest/reference_appendices/config.html#diff)

```bash
# 1. Validar sintaxis
ansible-playbook playbooks/day0-setup.yml --syntax-check

# 2. Dry-run + ver diferencias (DRY RUN marker)
ansible-playbook playbooks/day0-setup.yml --check --diff --limit servera

# 3. Ver diff real
ansible-playbook playbooks/day0-setup.yml --diff --limit servera

# 4. Producción completa
ansible-playbook playbooks/day0-setup.yml --diff
```

**Qué hace cada flag:**
- `--check` / `-C`: No hace cambios, predice qué haría. Marca `DRY RUN` al inicio/fin y `CHECK MODE` en cada task
- `--diff`: Muestra diff unificado de archivos (antes/después) para módulos que lo soportan (copy, template, lineinfile)
- `--limit servera`: Solo ejecuta en ese host - ideal para probar en 1 nodo antes de ir a todos
- Combinado: `ansible-playbook -i inventory/hosts site.yml --check --diff` - Tip oficial: *Test before production*

---

## 7. Linting y Validación

`ansible-lint` verifica sintaxis, malas prácticas y módulos deprecados.

**Referencia Oficial:**
- [Ansible Lint Documentation](https://ansible-lint.readthedocs.io/)
- [Find mistakes in your playbooks with Ansible Lint - Red Hat](https://www.redhat.com/en/blog/find-mistakes-ansible-lint)
- [Ansible Lint Rules](https://ansible-lint.readthedocs.io/rules/)

**Instalación:**

```bash
pip3 install ansible-lint
# o
dnf install ansible-lint

ansible-lint --version
```

**Uso en Sesión 1:**

```bash
# Lint a playbook
ansible-lint playbooks/day0-setup.yml

# Lint a todo el proyecto
ansible-lint

# Ver todas las reglas
ansible-lint -L

# Ejemplo salida:
# fqcn[action-core]: Use FQCN for builtin actions.
# yaml[indentation]: Wrong indentation
```

**Otras validaciones:**
```bash
ansible-playbook --syntax-check playbooks/day0-setup.yml
ansible-inventory --graph --vars
```

---

## 8. Lab Podman-Compose - 1 Controller + 5 Managed

Usamos Podman para simular datacenter sin necesidad de 6 VMs.

**Referencias:**
- [Podman Documentation](https://podman.io/docs)
- [podman-compose](https://github.com/containers/podman-compose)
- [Rocky Linux Docker Hub](https://hub.docker.com/r/rockylinux/rockylinux)

**`podman-compose.yml` usado en Sesión 1:**

```yaml
version: '3'
services:
  control:
    image: rockylinux:9
    container_name: alumno01_control
    hostname: control
    privileged: true
    command: /usr/sbin/init
    volumes:
      - ./playbooks:/home/ansible/playbooks:Z
    networks:
      telcel_lab:
        ipv4_address: 10.10.10.10

  servera:
    image: rockylinux:9
    container_name: alumno01_servera
    hostname: servera
    privileged: true
    command: /usr/sbin/init
    networks:
      telcel_lab:
        ipv4_address: 10.10.10.11
  # ... serverb-e similar

networks:
  telcel_lab:
    driver: bridge
    ipam:
      config:
        - subnet: 10.10.10.0/24
```

**Comandos lab:**

```bash
podman-compose up -d
podman ps
podman exec -it alumno01_control bash
su - ansible
cd /home/ansible/playbooks
ansible all -m ping
```

---

## 9. Playbook day0-setup.yml - MOTD, Timezone, Repos, Paquetes, Chrony

Playbook integrador de Sesión 1 que deja los 5 nodos listos.

**Módulos usados - Docs Oficiales:**

| Módulo | Doc |
|--------|-----|
| `ansible.builtin.copy` | https://docs.ansible.com/ansible/latest/collections/ansible/builtin/copy_module.html |
| `ansible.builtin.template` | https://docs.ansible.com/ansible/latest/collections/ansible/builtin/template_module.html |
| `ansible.builtin.dnf` | https://docs.ansible.com/ansible/latest/collections/ansible/builtin/dnf_module.html |
| `ansible.builtin.service` | https://docs.ansible.com/ansible/latest/collections/ansible/builtin/service_module.html |
| `community.general.timezone` | https://docs.ansible.com/ansible/latest/collections/community/general/timezone_module.html |
| `ansible.builtin.file` | https://docs.ansible.com/ansible/latest/collections/ansible/builtin/file_module.html |

**Ejemplo `day0-setup.yml` simplificado:**

```yaml
---
- name: Day0 - Setup base TELCEL
  hosts: all
  become: yes
  gather_facts: yes

  tasks:
    - name: Configurar MOTD
      copy:
        content: |
          Servidor {{ inventory_hostname }} - {{ ansible_facts['distribution'] }}
          Managed by Ansible - TELCEL
        dest: /etc/motd
        mode: '0644'

    - name: Configurar timezone
      community.general.timezone:
        name: America/Mexico_City

    - name: Instalar paquetes base
      ansible.builtin.dnf:
        name:
          - vim
          - htop
          - tmux
          - git
          - chrony
        state: present

    - name: Configurar chrony desde template
      template:
        src: templates/chrony.conf.j2
        dest: /etc/chrony.conf
        mode: '0644'
        backup: yes
      notify: restart chronyd

  handlers:
    - name: restart chronyd
      service:
        name: chronyd
        state: restarted
```

**Handlers:** Solo se ejecutan al final del play, una sola vez aunque 10 tasks hagan `notify`. Si la task no reporta `changed`, NO dispara handler.

---

## 10. Cheat Sheet Sesión 1

```bash
# Validación
ansible-config dump --only-changed
ansible-inventory --graph
ansible-inventory --host servera --yaml

# Modo seguro
ansible-playbook playbooks/day0-setup.yml --syntax-check
ansible-playbook playbooks/day0-setup.yml --check --diff --limit servera
ansible-playbook playbooks/day0-setup.yml --diff

# Lint
ansible-lint
ansible-lint playbooks/day0-setup.yml

# Lab
podman-compose up -d
podman ps
podman exec -it alumno01_control bash
su - ansible
ansible all -m ping -k   # primera vez con password linux123
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ""
for h in servera serverb serverc serverd servere; do ssh-copy-id ansible@$h; done
ansible all -m ping  # ya sin password

# Proyecto
cat ansible.cfg
cat inventory/hosts
cat playbooks/day0-setup.yml
```

---

## 📎 Referencias Generales Adicionales

- **Ansible Official Docs:** https://docs.ansible.com/
- **Ansible Best Practices - Red Hat:** https://www.redhat.com/en/topics/automation/what-is-an-ansible-playbook
- **12 Best Practices to Follow Before Running an Ansible Playbook on RHEL - Medium:** https://medium.com/@jerome.decinco/12-best-practices-to-follow-before-running-an-ansible-playbook-on-red-hat-enterprise-linux
- **Ansible Tips and Tricks:** https://docs.ansible.com/ansible/latest/tips_tricks/tips_tricks.html

---

**Autor:** TC Senior HPE | **Cliente:** TELCEL | **Fecha:** Septiembre 2026 | **Versión:** v1.0 | **Formato:** GitHub Markdown

> Learn more at Services | HPE | HPE GreenLake Portfolio | HPE
