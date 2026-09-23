# Sesión 2 - Referencias Bibliográficas Oficiales
## Workshop Ansible Intermedio - TELCEL
### Variables Inteligentes, Facts, group_vars/host_vars y Variables Mágicas

> **Cliente:** TELCEL | **Duración:** 3 Horas | **HPE A&PS** | Septiembre 2026

---

## 📚 Índice

1. [Variables y Precedencia](#1-variables)
2. [group_vars y host_vars](#2-group-host)
3. [Facts Nativos - ansible_facts](#3-facts)
4. [Custom Facts - /etc/ansible/facts.d](#4-custom-facts)
5. [Variables Mágicas](#5-magic)
6. [Validación e Inventario](#6-inventario)
7. [Cheat Sheet Sesión 2](#7-cheat)

---

## 1. Variables y Precedencia

**Concepto clave:** Host variables tienen mayor prioridad que group variables. Extra vars `-e` siempre ganan.

**Documentación Oficial:**

- [Understanding Variable Precedence - Ansible Docs](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_variables.html#understanding-variable-precedence)
  > Orden oficial de menor a mayor: role defaults → inventory group_vars → inventory host_vars → playbook group_vars → playbook host_vars → host facts → play vars → extra vars (always win)
- [Using Variables - Ansible Documentation](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_variables.html)

**Orden resumido (menor a mayor):**

```text
1. role defaults (roles/x/defaults/main.yml)
2. inventory file or script group vars
3. inventory group_vars/all
4. inventory group_vars/*
5. inventory host_vars/*
6. host facts / cached set_facts
7. play vars, vars_files, include_vars
8. role vars (roles/x/vars/main.yml)
9. block vars / task vars
10. extra vars (-e) -> siempre gana
```

**Ejemplo TELCEL:**
```bash
# Extra vars gana a todo
ansible-playbook playbooks/session2-variables.yml -e "http_port=9090"
```

> Ref: Extra vars (`-e`) always win - Host variables have higher priority than group variables[^1]

---

## 2. group_vars y host_vars

**Documentación Oficial:**

- [How to build your inventory - group_vars and host_vars](https://docs.ansible.com/ansible/latest/inventory_guide/intro_inventory.html#how-variables-are-merged)
- [Organizing host and group variables](https://docs.ansible.com/ansible/latest/inventory_guide/intro_inventory.html#organizing-host-and-group-variables)
- [Variable Precedence: Child group > Parent group > all](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html#how-variables-are-merged)

**Estructura oficial recomendada:**

```text
inventory/
├── hosts
├── group_vars/
│   ├── all.yml          # menos prioridad
│   ├── web.yml          # grupo web
│   └── db.yml           # grupo db
└── host_vars/
    ├── servera.yml      # más prioridad - solo servera
    └── serverd.yml
```

> Ref: Children have higher precedence than parent groups. Order: all group → parent group → child group → host[^2]

**Ejemplo TELCEL Sesión 2:**

```yaml
# inventory/group_vars/all.yml
ntp_server: mx.pool.ntp.org

# inventory/group_vars/web.yml
http_port: 8080

# inventory/host_vars/servera.yml
http_port: 8081  # gana - host_vars > group_vars
```

---

## 3. Facts Nativos - ansible_facts

**Documentación Oficial:**

- [Discovering variables: facts and magic variables](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_vars_facts.html)
- [Ansible facts - setup module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/setup_module.html)

**Uso:**

```bash
# Ver todos los facts
ansible servera -m setup

# Filtrar
ansible servera -m setup -a "filter=ansible_distribution*"

# En playbook
ansible_facts['distribution'] -> Rocky
ansible_facts['default_ipv4']['address'] -> 10.10.1.11
ansible_facts['memtotal_mb'] -> 2048
```

**Desactivar gather_facts:**
```yaml
- hosts: all
  gather_facts: no  # más rápido si no necesitas facts
```

---

## 4. Custom Facts - /etc/ansible/facts.d

**Documentación Oficial:**

- [Local facts - /etc/ansible/facts.d](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_vars_facts.html#adding-custom-facts)
- [setup module - fact_path parameter](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/setup_module.html#parameter-fact_path)

> Path used for local ansible facts (*.fact) - files in this dir will be run (if executable) and their results be added to ansible_local facts[^3]

**Formato:** INI o JSON, extensión `.fact`, se lee en `ansible_local`

**Ejemplo TELCEL:**

```ini
# /etc/ansible/facts.d/telcel.fact
[general]
telcel_env = prod
patch_level = 2025.09.15
datacenter = CDMX-01
```

```bash
ansible all -m setup -a "filter=ansible_local"
# Resultado:
# "ansible_local": {
#   "telcel": {
#     "general": {
#       "telcel_env": "prod",
#       "patch_level": "2025.09.15"
#     }
#   }
# }

ansible all -m debug -a "var=ansible_local.telcel.general.telcel_env"
```

**Uso en playbook:**

```yaml
- debug:
    msg: "Env {{ ansible_local.telcel.general.telcel_env }} patch {{ ansible_local.telcel.general.patch_level }}"
```

---

## 5. Variables Mágicas

**Documentación Oficial:**

- [Special Variables / Magic Variables - Ansible Documentation](https://docs.ansible.com/ansible/latest/reference_appendices/special_variables.html)
- [Discovering variables: facts and magic variables - Common magic variables](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_vars_facts.html#information-discovered-from-systems-facts)

> The most commonly used magic variables are `hostvars`, `groups`, `group_names`, and `inventory_hostname`. With `hostvars`, you can access variables defined for any host in the play[^4]

### Tabla oficial de mágicas:

| Variable | Descripción | Uso TELCEL |
|----------|-------------|------------|
| `inventory_hostname` | The inventory name for the 'current' host being iterated over in the play | Nombre lógico en inventory |
| `inventory_hostname_short` | The short version of inventory_hostname | Nombre corto sin dominio |
| `group_names` | List of groups the current host is part of | `when: 'web' in group_names` |
| `groups` | A dictionary/map with all the groups in inventory and each group has the list of hosts | `groups['db'][0]` primer DB |
| `hostvars` | A dictionary/map with all the hosts in inventory and variables assigned to them | `hostvars[groups['db'][0]]['ansible_facts']['default_ipv4']['address']` |
| `inventory_dir` | The directory of the inventory source | `{{ inventory_dir }}/files/` rutas relativas seguras |
| `inventory_file` | The file name of the inventory source | Debug qué inventory usas |
| `ansible_check_mode` | Boolean True si --check | `when: not ansible_check_mode` |
| `ansible_version` | Versión Ansible | Version check |
| `play_hosts` / `ansible_play_hosts` | Hosts del play actual | Loop sobre hosts del play |

[^5][^6]

### Casos de uso TELCEL con hostvars:

```yaml
# Web necesita IP de DB primario sin hardcodear
db_primary: "{{ groups['db'][0] }}"
db_ip: "{{ hostvars[db_primary]['ansible_facts']['default_ipv4']['address'] }}"

# Generar /etc/hosts dinámico
{% for host in groups['lab'] %}
{{ hostvars[host]['ansible_facts']['default_ipv4']['address'] }} {{ host }}
{% endfor %}
```

> Ref: hostvars - access variables and facts from other hosts. group_names - list of groups current host is in. groups - all groups (and hostnames) in inventory[^7]

---

## 6. Validación e Inventario

**Documentación Oficial:**

- [ansible-inventory command](https://docs.ansible.com/ansible/latest/cli/ansible-inventory.html)
- [Tips and tricks - inventory](https://docs.ansible.com/ansible/latest/tips_tricks/tips_tricks.html)

```bash
# Ver grafo
ansible-inventory --graph

# Ver variables efectivas de un host (precedencia aplicada)
ansible-inventory --host servera --yaml

# Ver groups y hostvars
ansible-inventory --list | jq '.web'
```

---

## 7. Cheat Sheet Sesión 2

```bash
# Precedencia
ansible-inventory --host servera --yaml | grep http_port
# host_vars gana a group_vars, extra vars -e gana a todo

# Facts
ansible all -m setup -a "filter=ansible_default_ipv4"
ansible all -m setup -a "filter=ansible_local"
ansible all -m debug -a "var=ansible_local.telcel.general.telcel_env"

# Custom facts
cat /etc/ansible/facts.d/telcel.fact
# [general]
# telcel_env = prod

# Mágicas
ansible all -m debug -a "var=inventory_hostname"
ansible all -m debug -a "var=group_names"
ansible all -m debug -a "var=groups"
ansible all -m debug -a "var=hostvars[groups['db'][0]]"

# Playbook con variables
ansible-playbook playbooks/session2-variables.yml --check --diff --limit servera
ansible-playbook playbooks/session2-magic-vars.yml --diff

# Validar IP cruzada web->db
ansible web -a "cat /tmp/magic-vars.conf"
```

---

## 📎 Referencias Generales Sesión 2

- **Ansible Docs - Using Variables:** https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_variables.html
- **Ansible Docs - Variable Precedence:** https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_variables.html#understanding-variable-precedence
- **Ansible Docs - Inventory - Organizing variables:** https://docs.ansible.com/ansible/latest/inventory_guide/intro_inventory.html
- **Ansible Docs - Facts and Magic Variables:** https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_vars_facts.html
- **Ansible Docs - Special Variables (Magic):** https://docs.ansible.com/ansible/latest/reference_appendices/special_variables.html
- **Ansible Docs - setup module - Local facts:** https://docs.ansible.com/ansible/latest/collections/ansible/builtin/setup_module.html

---

**Autor:** TC Senior HPE | **Cliente:** TELCEL | **Fecha:** Septiembre 2026 | **Sesión 2:** Variables Inteligentes, Facts, group_vars/host_vars y Variables Mágicas

[^1]: bilalbayasut/seal - docs/configuration-management/ansible-variables.md — https://github.com/bilalbayasut/seal/blob/HEAD/docs/configuration-management/ansible-variables.md
[^2]: Child group variables do not override parent group variables in `group_vars` folder — https://github.com/ansible/ansible/issues/56127
[^3]: ansible.builtin.setup – Gathers facts about remote hosts — Ansible Documentation — https://docs.ansible.com/ansible/2.10/collections/ansible/builtin/setup_module.html
[^4]: Discovering variables: facts and magic variables — Ansible Community Documentation — https://docs.ansible.com/ansible/10/playbook_guide/playbooks_vars_facts.html
[^5]: natrontech/workshop-ansible - docs/basics/inventory/variables.md — https://github.com/natrontech/workshop-ansible/blob/HEAD/docs/basics/inventory/variables.md
[^6]: Special Variables — Ansible Documentation — https://docs.ansible.com/ansible/2.8/reference_appendices/special_variables.html
[^7]: groovemonkey/hands-on-ansible - cheat_sheets/variables.md — https://github.com/groovemonkey/hands-on-ansible/blob/HEAD/cheat_sheets/variables.md
