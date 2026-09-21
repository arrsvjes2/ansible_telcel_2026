# Workshop Ansible Intermedio - TELCEL
**HPE Advisory & Professional Services | 9 Sesiones x 3 Horas | 27 Horas Totales**

Laboratorio práctico con Podman - 1 Control + 5 Nodos Managed (servera-e)

---

## 📋 Descripción

Workshop intermedio de Ansible para SysAdmins, SREs y Admins Linux de TELCEL. Enfoque 100% práctico con principio DRY, ejecución segura `--check --diff --limit` y buenas prácticas HPE.

- **Cliente:** TELCEL
- **Duración:** 27 horas (9 sesiones x 3h)
- **Fecha inicio:** 27 Septiembre 2026
- **Lab:** Podman Compose - Rocky 9 / RHEL
- **Usuario lab:** `ansible / linux123`

---

## 🗂️ Estructura del Proyecto

```text
ansible-telcel/
├── README.md
├── ansible.cfg
├── inventory/
│   └── hosts
├── group_vars/
│   ├── all.yml
│   ├── web.yml
│   └── db.yml
├── host_vars/
│   ├── servera.yml
│   └── servere.yml
├── playbooks/
│   ├── day0-setup.yml
│   ├── session2-variables.yml
│   ├── session3-sshd-hardening.yml
│   ├── session3-logrotate.yml
│   ├── session4-user-management.yml
│   └── session4-package-audit.yml
├── templates/
│   ├── sshd_config.j2
│   ├── chrony.conf.j2
│   └── logrotate-telcel.j2
└── docs/
    ├── sesion1-fundamentos.md
    ├── sesion2-variables-facts.md
    └── referencias.md
```

---

## 🚀 Quick Start

### 1. Levantar lab

```bash
podman-compose up -d
podman ps
```

### 2. Entrar al control

```bash
podman exec -it ansible_control_1 bash
su - ansible
cd /home/ansible/playbooks
```

### 3. Configurar SSH

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ""
for h in servera serverb serverc serverd servere; do ssh-copy-id ansible@$h; done
ansible all -m ping
```

### 4. Ejecutar playbooks (modo seguro)

```bash
# Validar
ansible-playbook playbooks/day0-setup.yml --syntax-check
ansible-playbook playbooks/day0-setup.yml --check --diff --limit servera

# Ejecutar
ansible-playbook playbooks/day0-setup.yml --diff
```

---

## 📚 Sesiones

| # | Sesión | Tema | Duración |
|---|--------|------|----------|
| 1 | Fundamentos | Arquitectura, Estructura, --check --diff --limit, ansible-lint, Lab Podman | 3h |
| 2 | Variables y Facts | Precedencia all->group->host, group_vars/host_vars, ansible_facts, custom facts /etc/ansible/facts.d/ | 3h |
| 3 | Tasks, Handlers y Notify | Tasks atómicas, handlers, changed_when, Hardening SSH + logrotate | 3h |
| 4 | Loops y Condicionales | loop, when, register, dict2items, gestión usuarios + auditoría paquetes | 3h |
| 5 | Ejecución Controlada | tags, block/rescue/always, estrategias, limit | 3h |
| 6 | Templates Jinja2 | Templates dinámicos, filtros, validación | 3h |
| 7 | Roles | Roles, ansible-galaxy, estructura roles | 3h |
| 8 | Vault y Seguridad | Ansible Vault, manejo secretos | 3h |
| 9 | Proyecto Integrador | Proyecto final + troubleshooting | 3h |

---

## 🛠️ Requisitos

- Podman / Docker + podman-compose
- Rocky 9 / RHEL 9
- Python 3.9+
- ansible-core 2.14+
- ansible-lint

```bash
pip install ansible ansible-lint
```

---

## 📎 Docs

- [Sesión 1 - Fundamentos y Lab](docs/Ansible_Sesion1_TELCEL_GitHub.md)
- [Sesión 1 - Referencias Oficiales](docs/SESION_1_Referencias_GitHub.md)
- [Sesión 2 - Variables y Facts](ansible-sesion2/SESION_2_Variables_Facts_TELCEL.docx)
- [Sesión 3 - Tasks y Handlers](ansible-sesion3/SESION_3_Tasks_Handlers_TELCEL.docx)
- [Sesión 4 - Loops y Condicionales](ansible-sesion4/SESION_4_Loops_Condicionales_TELCEL.docx)

Documentación oficial: https://docs.ansible.com/

---

## 👨‍💻 Autor

TC Senior HPE | TELCEL | Septiembre 2026

> DRY - Don't Repeat Yourself. Un playbook, muchas variables.
