# Sesión 6 Completa - Tags + Control Errores | TELCEL | HPE A&PS
## Tags (--tags, --skip-tags) 1h + block/rescue/always 2h | 3 Horas | 100% Offline

**Instructor:** Tú | **Cliente:** TELCEL | **Lab:** Containers sin internet ni init | **Ansible:** 2.16.16

---

## Objetivo
Usar tags para ejecución selectiva y controlar errores con block/rescue/always, failed_when, ignore_errors. 100% offline.

---

## 1. Teoría Tags (1h)

### 1.1 Qué son Tags
Tags filtran qué corre. En TELCEL 100 servidores, solo cambiar app.conf en 1: `--tags config --limit servera` 30 seg vs site.yml completo 10 min.

### 1.2 Sintaxis
```yaml
- template: src=app.conf.j2 dest=/etc/telcel/app.conf
  tags: [config, app, deploy]

- debug: msg="Siempre"
  tags: [always]

- debug: var=hostvars[inventory_hostname]
  tags: [never, debug]
```

En role:
```yaml
- { role: telcel_app, tags: [app, deploy] }
```

Especiales: `always` siempre corre, `never` solo con `--tags never`. `tagged` solo con tags, `untagged` solo sin tags.

### 1.3 Comandos
```bash
--list-tags --list-tasks
--tags config
--tags app,web
--skip-tags logs
--tags always,config
--tags never,debug
--tags tagged
--tags untagged
```

Buena práctica TELCEL: capas web, db, app, config, deploy, logs, validate. always validaciones, never debug pesado.

---

## 2. Teoría Control Errores (2h)

### 2.1 ignore_errors
```yaml
- command: cat /noexiste.conf
  ignore_errors: true
```

### 2.2 failed_when
```yaml
- shell: grep -q app_port /etc/telcel/app.conf && echo OK || echo FAIL
  register: check
  failed_when: "'FAIL' in check.stdout"
```

### 2.3 changed_when
```yaml
- command: cat /etc/telcel/app.conf
  changed_when: false
```

### 2.4 any_errors_fatal / max_fail_percentage
```yaml
- hosts: web
  any_errors_fatal: true

- hosts: web
  max_fail_percentage: 25
```

### 2.5 block/rescue/always - Corazón
```yaml
- block:
    - template: src=app.conf.j2 dest=/etc/telcel/app.conf.new
    - shell: grep -q app_port /etc/telcel/app.conf.new
    - copy: src=/etc/telcel/app.conf.new dest=/etc/telcel/app.conf remote_src=yes
  rescue:
    - file: path=/etc/telcel/app.conf.new state=absent
    - copy: content="app_port=8080" dest=/etc/telcel/app.conf
  always:
    - command: cat /etc/telcel/app.conf
```

block try, rescue rollback, always auditoría. Evita dejar host sin config.

---

## 3. LAB 6.1 - Tags TELCEL (1h) Offline

**playbooks/session6-tags.yml**
```yaml
---
- hosts: all
  tasks:
    - debug: msg="Always"
      tags: [always]
    - file: path=/etc/telcel state=directory
      tags: [always, deploy, config]
    - template: src=telcel-app.conf.j2 dest=/etc/telcel/app.conf validate=sh -c "grep -q 'app_port' %s"
      tags: [config, app, deploy]
    - template: src=telcel-httpd.conf.j2 dest=/etc/httpd/conf.d/telcel.conf validate=sh -c "grep -q 'Listen' %s"
      tags: [config, web, deploy]
    - template: src=logrotate-telcel.j2 dest=/etc/logrotate.d/telcel validate=sh -c "grep -q 'daily' %s"
      tags: [logs, config]
    - debug: var=hostvars[inventory_hostname]
      tags: [never, debug]
```

```bash
ansible-playbook playbooks/session6-tags.yml --list-tags
ansible-playbook playbooks/session6-tags.yml --tags config --diff --limit servera
ansible-playbook playbooks/session6-tags.yml --skip-tags logs --diff
ansible-playbook playbooks/session6-tags.yml --tags never,debug --diff
ansible-playbook playbooks/session6-tags.yml --diff # 2da 0 changed
```

Ejercicio: Crea 5 tags base, config, deploy, logs, validate, always, never. Prueba --tags config y --skip-tags logs.

---

## 4. LAB 6.2 - Control Errores block/rescue/always (1h) Offline

**playbooks/session6-errors.yml**
```yaml
---
- hosts: web
  tasks:
    - block:
        - command: cat /noexiste.conf
          ignore_errors: true
      tags: [ignore_errors]

    - block:
        - shell: grep -q app_port /etc/telcel/app.conf && echo OK || echo FAIL
          register: check
          failed_when: "'FAIL' in check.stdout"
      rescue:
        - copy: content="app_port=8080" dest=/etc/telcel/app.conf
      tags: [failed_when]

    - block:
        - template: src=telcel-app.conf.j2 dest=/etc/telcel/app.conf.new validate=sh -c "grep -q 'app_port' %s"
        - shell: grep -q app_port /etc/telcel/app.conf.new
        - copy: src=/etc/telcel/app.conf.new dest=/etc/telcel/app.conf remote_src=yes
      rescue:
        - file: path=/etc/telcel/app.conf.new state=absent
        - copy: content="app_port=8080" dest=/etc/telcel/app.conf
      always:
        - command: cat /etc/telcel/app.conf
          changed_when: false
      tags: [block]
```

```bash
ansible-playbook playbooks/session6-errors.yml --tags block --diff --limit servera
ansible-playbook playbooks/session6-errors.yml --tags block --diff -e "app_port=FAIL" # Fuerza rescue
```

Ejercicio: block cat missing ignore_errors, block grep failed_when rescue crea básico, block template rescue fallback always cat.

---

## 5. LAB 6.3 - Integrado Tags + Errores + Roles (1h)

**playbooks/session6-integrado.yml**
```yaml
---
- hosts: web
  tasks:
    - debug: msg="Inicio"
      tags: [always]

    - block:
        - template: src=telcel-app.conf.j2 dest=/etc/telcel/app.conf validate=sh -c "grep -q 'app_port' %s" backup=yes
          tags: [config, deploy]
        - template: src=telcel-httpd.conf.j2 dest=/etc/httpd/conf.d/telcel.conf validate=sh -c "grep -q 'Listen' %s" backup=yes
          tags: [config, deploy]
        - shell: grep -q app_port /etc/telcel/app.conf && echo "DEPLOY OK"
          failed_when: "'OK' not in deploy_valid.stdout"
          tags: [validate]
      rescue:
        - shell: cp /etc/telcel/app.conf.bak /etc/telcel/app.conf || echo "app_port=8080" > /etc/telcel/app.conf
      always:
        - shell: cat /etc/telcel/app.conf
          changed_when: false
      tags: [deploy]

    - template: src=logrotate-telcel.j2 dest=/etc/logrotate.d/telcel
      tags: [logs]

    - debug: var=hostvars[inventory_hostname]
      tags: [never, debug]
```

```bash
ansible-playbook playbooks/session6-integrado.yml --tags config --diff --limit servera
ansible-playbook playbooks/session6-integrado.yml --tags deploy --diff
ansible-playbook playbooks/session6-integrado.yml --skip-tags logs --diff
ansible-playbook playbooks/session6-integrado.yml --tags never --diff
ansible-playbook playbooks/session6-integrado.yml --diff # 2da 0 changed
```

---

## 6. Reto Final

Crear session6-reto.yml con:
- always: dirs
- config/deploy: block template validate grep + rescue rollback + always cat
- logs: logrotate
- never: debug hostvars + ls -R
- handler listen reload app

Probar:
```bash
--list-tags --tags config --tags deploy --skip-tags logs --tags never --diff
# 2da 0 changed
# -e "app_port=" fuerza rescue
```

---

## 7. Soluciones y Validación

Solución reto ejemplo en docx completo. Validación:

```bash
ansible-playbook playbooks/session6-reto.yml --diff --limit servera # 1ra changed
ansible-playbook playbooks/session6-reto.yml --diff --limit servera # 2da 0 changed = OK
--tags config # Solo config
--tags never # Solo never
-e "app_port=" # Fuerza rescue
```

---

## 8. Cheat Sheet

```bash
--list-tags --list-tasks --tags config --tags app,web --skip-tags logs --tags always,config --tags never,debug --tags tagged --tags untagged
```

```yaml
tags: [config, deploy] / tags: always / tags: never / tags: [never, debug]
ignore_errors: true
failed_when: "'FAIL' in result.stdout" / failed_when: result.rc != 0
changed_when: false
any_errors_fatal: true / max_fail_percentage: 25
block/rescue/always
validate: sh -c "grep -q 'app_port' %s"
```

---

## 9. Anexos Templates Offline

**telcel-app.conf.j2**
```
app_host={{ inventory_hostname }}
app_port={{ app_port }}
app_env={{ app_env }}
```

**telcel-httpd.conf.j2**
```
Listen {{ app_port }}
ServerName {{ inventory_hostname }}.telcel.lab
DocumentRoot /var/www/telcel
```

**logrotate-telcel.j2**
```
{{ log_path }} {
    daily
    rotate {{ log_rotate }}
    postrotate
        echo "Rotado {{ inventory_hostname }}" > /tmp/logrotate-done
    endscript
}
```

**ansible.cfg**
```ini
[defaults]
inventory = inventory/hosts
roles_path = ./roles
vault_password_file = ~/.vault_pass
```

Todo offline sin dnf, service, httpd -t, solo grep/cat.

**Autor:** HPE A&PS | TELCEL | Sesión 6 | Sept 2026 | Documento Completo
