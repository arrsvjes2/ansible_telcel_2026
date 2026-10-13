# Sesión 8 - Ansible Vault en Playbooks - TELCEL Lab
## Rocky Linux 10.1 + Ansible 2.16.16

---

## 1. Definición de Datos Sensibles

**Dato sensible:** Información que si se expone compromete seguridad, cumplimiento o negocio. **NUNCA** debe ir en git en texto plano.

### Ejemplos TELCEL (tu lab)

| Tipo | Ejemplo | Riesgo si se filtra |
|------|---------|---------------------|
| Passwords DB | `db_password: Telcel2024_S3cr3t!` | Acceso total a BD prod |
| Tokens API | `api_token: eyJhbGciOiJI...` | Suplantación servicio |
| Llaves SSL privada | `-----BEGIN PRIVATE KEY-----` | Decrypt tráfico, MITM |
| Secret Key App | `SECRET_KEY: django-insecure-...` | Firma sesiones, bypass auth |
| AWS Keys | `aws_secret_access_key: wJalr...` | Facturación, borrado infra |
| Ansible become pass | `ansible_become_password: ...` | Root en 1000 hosts |
| Certificados | `vault_ssl_cert: ...` |  |

**Regla TELCEL:**
```
NO sensible → group_vars/web/vars.yml (git)
Sensible → group_vars/web/vault.yml (cifrado con vault, git ok porque cifrado)
```

### Ejemplo Real Sin Vault (MAL - Sesión 6)

```yaml
# group_vars/web.yml - MAL, password en texto plano en git
db_user: telcel_app
db_password: P@ssw0rd123  # ← SI ALGUIEN HACE git log, VE ESTO
```

### Con Vault (BIEN - Sesión 8)

```yaml
# group_vars/web/vars.yml - NO sensible, en git plano
db_user: telcel_app
db_host: db.telcel.lab

# group_vars/web/vault.yml - SENSIBLE, cifrado, en git pero cifrado
$ANSIBLE_VAULT;1.1;AES256
313462663933383065613666623... # db_password cifrado
```

---

## 2. ¿En Qué Consiste Ansible Vault?

Vault es cifrado **AES256** integrado en Ansible para archivos y variables.

**Cómo funciona:**

```
Texto plano → AES256 + password → $ANSIBLE_VAULT;1.1;AES256...
AES256 vault → password → texto plano (en memoria, nunca en disco plano durante run)

Flujo:
1. Escribes vault.yml plano
2. ansible-vault encrypt vault.yml → cifra archivo
3. git add vault.yml cifrado (seguro)
4. ansible-playbook --ask-vault-pass → descifra en RAM, usa var, no deja rastro en disco
```

**No es:**
- No es gestor de secretos (HashiCorp Vault, AWS Secrets Manager) - es cifrado local
- No cifra en tránsito (usa SSH para eso)
- No cifra logs si no usas `no_log: true`

---

## 3. ansible-vault - Comandos

### 3.1 Crear archivo cifrado nuevo

```bash
cd ~/taller-ansible-telcel

# Crea group_vars/web/vault.yml cifrado desde cero (abre editor)
ansible-vault create group_vars/web/vault.yml

# Te pide: New Vault password: ****
# Te abre $EDITOR (vi), escribes:
---
db_password: "Telcel2024_S3cur3!"
api_token: "token-prod-telcel-xyz"
ssl_key_pass: "s3cr3t"

# Guardas :wq, queda cifrado
cat group_vars/web/vault.yml
# $ANSIBLE_VAULT;1.1;AES256
# 6638346231333565303062366562...
```

### 3.2 Ver archivo cifrado

```bash
ansible-vault view group_vars/web/vault.yml
# Vault password: ****
# Muestra contenido descifrado en pantalla, no lo descifra en disco

# Con archivo de password
ansible-vault view --vault-password-file ~/.vault_pass group_vars/web/vault.yml
```

### 3.3 Editar archivo cifrado

```bash
ansible-vault edit group_vars/web/vault.yml
# Vault password:
# Abre editor con contenido descifrado, al guardar recifra automático

# Editar con pass file
ansible-vault edit --vault-password-file ~/.vault_pass group_vars/web/vault.yml
```

### 3.4 Cifrar archivo existente plano

```bash
# Ya tienes group_vars/web/vault.yml plano
cat group_vars/web/vault.yml
# db_password: plain

ansible-vault encrypt group_vars/web/vault.yml
# Vault password: ****
# Ahora cat muestra $ANSIBLE_VAULT...

# Cifrar con pass file
ansible-vault encrypt --vault-password-file ~/.vault_pass group_vars/web/vault.yml

# Cifrar string suelto (para poner en yaml)
ansible-vault encrypt_string 'Telcel2024_S3cur3!' --name 'db_password'
# Te pide password vault
# Salida:
# db_password: !vault |
#   $ANSIBLE_VAULT;1.1;AES256
#   633833....

# Útil para pegar en group_vars/web/vars.yml sin archivo aparte
```

### 3.5 Descifrar archivo (a plano) - CUIDADO

```bash
# Descifra a plano en disco (no recomendado en prod, solo debug)
ansible-vault decrypt group_vars/web/vault.yml
# Vault password:
# Ahora queda plano en disco - recifra después!

# Para ver sin descifrar en disco, usa view, no decrypt
```

### 3.6 Recifrar con nuevo password

```bash
ansible-vault rekey group_vars/web/vault.yml
# Old password:
# New password:
```

---

## 4. Uso con group_vars y host_vars (Recomendado TELCEL)

Ansible carga automáticamente `group_vars/` y `host_vars/` si existen.

### Estructura Sesión 8

```
taller-ansible-telcel/
├── inventory.ini
├── group_vars/
│   ├── all/
│   │   ├── vars.yml          # vars comunes plano
│   │   └── vault.yml         # vault all (cifrado)
│   └── web/
│       ├── vars.yml          # plano - db_user, app_port
│       └── vault.yml         # cifrado - db_password, api_token
├── host_vars/
│   ├── servera/
│   │   └── vault.yml         # cifrado solo para servera
│   └── serverb.yml           # plano
└── playbooks/
    └── session8-vault.yml
```

### Ejemplo group_vars/web/vars.yml (plano)

```yaml
---
db_user: telcel_app
db_host: db.telcel.lab
db_name: telcel_lab
app_port: 8080
```

### Ejemplo group_vars/web/vault.yml (antes de cifrar)

```yaml
---
db_password: "Telcel2024_S3cur3!"
api_token: "eyJhbGciOiJIUzI1NiJ9.telcel"
```

### Cifrarlo

```bash
ansible-vault encrypt group_vars/web/vault.yml
# Ahora git add group_vars/web/vault.yml es seguro
```

### Ansible lo carga automático - No necesitas vars_files

```yaml
# playbooks/session8-autoload.yml
---
- hosts: web
  tasks:
    - debug:
        var: db_user      # viene de group_vars/web/vars.yml
    - debug:
        var: db_password  # viene de group_vars/web/vault.yml descifrado en RAM
      no_log: true
```

Ejecuta:

```bash
ansible-playbook playbooks/session8-autoload.yml --ask-vault-pass
```

---

## 5. Uso en Tasks y Playbooks

### 5.1 En playbook con vars_files

```yaml
---
- hosts: web
  vars_files:
    - ../group_vars/web/vault.yml
  tasks:
    - name: Template con password vault
      template:
        src: telcel-app.conf.j2
        dest: /etc/telcel/app.conf
        mode: '0600'  # Solo root
      no_log: true  # No muestra password en log
```

### 5.2 En template

```jinja2
# templates/telcel-app.conf.j2
[database]
db_user={{ db_user }}
db_password={{ db_password }}  # viene de vault.yml
db_host={{ db_host }}

[api]
token={{ api_token }}
```

### 5.3 Encrypt_string en mismo archivo (sin archivo vault separado)

```bash
ansible-vault encrypt_string 'SuperSecret123!' --name 'db_password'
```

Pega salida en `group_vars/web/vars.yml`:

```yaml
---
db_user: telcel_app
db_password: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  633438323136653136613631353136323133...
```

### 5.4 no_log: true - OBLIGATORIO con vault

```yaml
- name: Usar password
  shell: echo {{ db_password }}
  no_log: true  # Si no, password sale en stdout de ansible

- name: Debug seguro
  debug:
    msg: "Conectando a DB"
  # Nunca: debug: var=db_password
```

---

## 6. Opciones ansible-playbook para Password Vault

### 6.1 Interactiva --ask-vault-pass (sesión 8 lab)

```bash
ansible-playbook playbooks/session8-vault.yml --ask-vault-pass
# Vault password: **** (teclado, no se ve)
```

**Uso:** Lab, debug, 1 host

### 6.2 Archivo de password --vault-password-file (recomendado CI/CD TELCEL)

```bash
# 1. Crea archivo con password (chmod 600)
echo "MiPasswordVault2024" > ~/.vault_pass
chmod 600 ~/.vault_pass
echo "~/.vault_pass" >> .gitignore

# 2. Usa en playbook
ansible-playbook playbooks/session8-vault.yml --vault-password-file ~/.vault_pass

# 3. Para vault create/edit/view
ansible-vault create --vault-password-file ~/.vault_pass group_vars/web/vault.yml
ansible-vault view --vault-password-file ~/.vault_pass group_vars/web/vault.yml
```

**Producción TELCEL:**

```bash
# En servidor CI/CD
echo "$VAULT_PASSWORD" > /etc/ansible/vault_pass
chmod 600 /etc/ansible/vault_pass

# ansible.cfg
[defaults]
vault_password_file = /etc/ansible/vault_pass
```

`ansible.cfg`:

```ini
[defaults]
vault_password_file = ~/.vault_pass
# o
vault_password_file = /etc/ansible/vault_pass_prod
```

Ahora `ansible-playbook` no necesita flag, lo lee automático.

### 6.3 Variable de ambiente ANSIBLE_VAULT_PASSWORD_FILE (TELCEL prod)

```bash
# Exporta ruta a archivo
export ANSIBLE_VAULT_PASSWORD_FILE=~/.vault_pass
ansible-playbook playbooks/session8-vault.yml

# O script que devuelve password (para HashiCorp Vault, AWS)
cat > ~/vault_pass.sh <<'EOF'
#!/bin/bash
echo "MiPassword2024"
# O: vault kv get -field=password secret/ansible
# O: aws secretsmanager get-secret-value --secret-id ansible/vault
EOF
chmod +x ~/vault_pass.sh
export ANSIBLE_VAULT_PASSWORD_FILE=~/vault_pass.sh

ansible-playbook playbooks/session8-vault.yml
```

### 6.4 Múltiples vault passwords (vault-id) - Avanzado

```bash
# Para prod y lab con passwords distintos
ansible-vault create --vault-id prod@~/.vault_prod group_vars/prod/vault.yml
ansible-vault create --vault-id lab@~/.vault_lab group_vars/lab/vault.yml

# Ejecuta con ambos
ansible-playbook playbooks/site.yml --vault-id prod@~/.vault_prod --vault-id lab@~/.vault_lab

# O en ansible.cfg
[defaults]
vault_identity_list = prod@~/.vault_prod, lab@~/.vault_lab
```

---

## 7. Labs Sesión 8 - Paso a Paso en Control

### Lab 8.1 - Crear vault

```bash
cd ~/taller-ansible-telcel
mkdir -p group_vars/web host_vars/servera

cat > group_vars/web/vars.yml <<'EOF'
---
db_user: telcel_app
db_host: db.telcel.lab
app_port: 8080
EOF

# Crea vault.yml cifrado
ansible-vault create group_vars/web/vault.yml
# Password: telcel123
# Contenido:
---
db_password: "Telcel2024_Lab!"
api_token: "lab-token-xyz"

# Verifica cifrado
cat group_vars/web/vault.yml
# $ANSIBLE_VAULT;1.1;AES256...

# Ver sin descifrar en disco
ansible-vault view group_vars/web/vault.yml
# Password: telcel123
```

### Lab 8.2 - Encrypt_string

```bash
ansible-vault encrypt_string 'ProdPass2024!' --name 'db_password'
# Password: telcel123
# Copia salida a group_vars/web/vars.yml
```

### Lab 8.3 - Ejecutar en control local (Sesión 7 + 8)

```bash
# inventory-control.ini
[control]
localhost ansible_connection=local ansible_python_interpreter=/usr/bin/python3.12

# ansible.cfg con vault
cat > ansible.cfg <<'EOF'
[defaults]
inventory = ./inventory-control.ini
roles_path = ./roles
vault_password_file = ~/.vault_pass
stdout_callback = default
EOF

echo "telcel123" > ~/.vault_pass
chmod 600 ~/.vault_pass

ansible-playbook playbooks/session8-autoload.yml --diff

# Interactivo
ansible-playbook playbooks/session8-autoload.yml --ask-vault-pass --diff

# Con env var
ANSIBLE_VAULT_PASSWORD_FILE=~/.vault_pass ansible-playbook playbooks/session8-autoload.yml --diff
```

### Lab 8.4 - Verificación no_log

```bash
# Sin no_log: password sale en log (MAL)
ansible-playbook playbooks/session8-vault.yml --vault-password-file ~/.vault_pass -v
# TASK [debug var=db_password] -> muestra password!

# Con no_log: true (BIEN)
# En task pon no_log: true
```

---

## 8. Checklist Sesión 8

- [ ] Identificas datos sensibles (passwords, tokens, keys)
- [ ] Creas `group_vars/web/vault.yml` con `ansible-vault create`
- [ ] `ansible-vault view` y `edit` funcionan
- [ ] `encrypt_string` para var suelta
- [ ] `group_vars/web/vars.yml` plano + `vault.yml` cifrado, ambos en git
- [ ] Playbook usa `no_log: true` con vault vars
- [ ] Ejecutas con `--ask-vault-pass`
- [ ] Ejecutas con `--vault-password-file ~/.vault_pass`
- [ ] `ansible.cfg` con `vault_password_file = ~/.vault_pass`
- [ ] `ANSIBLE_VAULT_PASSWORD_FILE=~/.vault_pass ansible-playbook ...`
- [ ] `~/.vault_pass` tiene `chmod 600` y está en `.gitignore`

---

## 9. Qué NO hacer

```bash
# ❌ NO subir vault_pass a git
git add ~/.vault_pass  # NUNCA

# ❌ NO hacer decrypt y dejar plano en disco
ansible-vault decrypt group_vars/web/vault.yml
git add group_vars/web/vault.yml  # Ahora subes plano!

# ❌ NO debug var sensible sin no_log
- debug: var=db_password  # Sale en log

# ❌ NO password 123456
# Usa: pwgen 32 -y o openssl rand -base64 32
```

---

> Fin Sesión 8 - Vault - TELCEL Lab


---

## 10. Anexos - Archivos Reales del Lab (copiar/pegar directo)

### 10.1 Estructura completa para copiar

```bash
cd ~/taller-ansible-telcel
mkdir -p group_vars/web host_vars/servera playbooks
```

### 10.2 group_vars/web/vars.yml (PLANO)

```yaml
---
# group_vars/web/vars.yml - Variables NO sensibles - va a git plano
db_user: telcel_app
db_host: db.telcel.lab
db_name: telcel_lab
db_port: 5432
app_port: 8080
app_env: lab
httpd_port: 8080
log_level: INFO
```

### 10.3 group_vars/web/vault.yml (ANTES de cifrar - luego ansible-vault encrypt)

```yaml
---
# group_vars/web/vault.yml - ESTE ARCHIVO SE CIFRA
# Antes de cifrar se ve así, después: $ANSIBLE_VAULT;1.1;AES256...
db_password: "Telcel2024_S3cur3_Lab!"
api_token: "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.telcel-lab-token"
ssl_key_pass: "T3lc3l_L4b_2024"
aws_secret_key: "wJalrXUtnFEMI/K7MDENG/bPxRfiCYSECRETKEY"
```

```bash
# Cifrarlo:
ansible-vault encrypt group_vars/web/vault.yml
# Password: telcel123
```

### 10.4 host_vars/servera/vault.yml (solo servera)

```yaml
---
# host_vars/servera/vault.yml - cifrado solo para servera
servera_ssl_cert_key: "-----BEGIN PRIVATE KEY----- servera only"
servera_api_secret: "servera-secret-xyz"
```

### 10.5 group_vars/all/vault.yml (para todos los hosts)

```yaml
---
# group_vars/all/vault.yml - secrets comunes a todos
ansible_become_password: "RootTelcel2024!"
vault_global_api_key: "global-key-telcel"
```

### 10.6 Playbook Sesión 8 - Autoload group_vars (recomendado)

```yaml
---
# playbooks/session8-autoload.yml
# No necesita vars_files, Ansible carga group_vars/web/* automático
- name: Sesion 8 - Vault autoload group_vars
  hosts: web
  become: true

  tasks:
    - name: Debug var NO sensible
      ansible.builtin.debug:
        var: db_user

    - name: Usar var vault en template (con no_log)
      ansible.builtin.template:
        src: telcel-app.conf.j2
        dest: /tmp/telcel-app-vault.conf
        mode: '0600'
      no_log: true

    - name: Crear config con password vault
      ansible.builtin.copy:
        content: |
          db_user={{ db_user }}
          db_password={{ db_password }}
          token={{ api_token }}
        dest: /tmp/db.conf
        mode: '0600'
      no_log: true
```

### 10.7 Playbook con vars_files explícito

```yaml
---
# playbooks/session8-vault.yml
- name: Sesion 8 - Vault con vars_files
  hosts: web
  become: true
  vars_files:
    - ../group_vars/web/vars.yml
    - ../group_vars/web/vault.yml

  tasks:
    - name: Template telcel-app con vault
      ansible.builtin.template:
        src: telcel-app.conf.j2
        dest: /etc/telcel/app.conf
        mode: '0600'
      no_log: true

    - name: Validar sin exponer password
      ansible.builtin.shell:
        cmd: grep -q db_user /etc/telcel/app.conf && echo OK
      changed_when: false
```

### 10.8 Template con variables vault

```jinja2
# roles/telcel_app/templates/telcel-app.conf.j2 - con vault
# {{ ansible_managed }}
[database]
db_user={{ db_user }}
db_password={{ db_password }}
db_host={{ db_host }}
db_name={{ db_name }}

[api]
token={{ api_token }}
env={{ app_env }}

[ssl]
key_pass={{ ssl_key_pass | default('') }}
```

### 10.9 Encrypt_string - var vault dentro de vars.yml plano

```bash
ansible-vault encrypt_string 'ProdSuperSecret2024!' --name 'db_password'
```

Salida para pegar en `group_vars/web/vars.yml`:

```yaml
---
db_user: telcel_app
db_password: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  663834623133356530306236656261653831646638353865663033656238386236
  6265303039626637323331313438343637396531623563360a3064383634303138
  3534646266636338366539373138316238646531646230653938663930336130
  6565326161366238666130313262646238316565373734336363360a66326164
  6238316565373734336363360a
```

### 10.10 ansible.cfg Sesión 8

```ini
[defaults]
inventory = ./inventory.ini
roles_path = ./roles
host_key_checking = False
interpreter_python = auto_silent
vault_password_file = ~/.vault_pass
stdout_callback = default

# Para múltiples vault ids
# vault_identity_list = prod@~/.vault_prod, lab@~/.vault_lab
```

### 10.11 .gitignore para vault

```
# .gitignore TELCEL
.vault_pass
*.vault_pass
vault_pass.txt
.vault/
group_vars/**/vault.yml.plain
```

### 10.12 Comandos completos Sesión 8 - Copy/Paste para control local

```bash
# --- Setup lab ---
cd /root
mkdir -p sesion8-lab/group_vars/web sesion8-lab/playbooks
cd sesion8-lab

cat > group_vars/web/vars.yml <<'EOF'
---
db_user: telcel_app
db_host: db.telcel.lab
app_port: 8080
EOF

# Crea vault.yml plano temporal
cat > group_vars/web/vault.yml <<'EOF'
---
db_password: "Telcel2024_Lab!"
api_token: "lab-token-xyz"
EOF

# Password file
echo "telcel123" > ~/.vault_pass
chmod 600 ~/.vault_pass

# Cifra vault.yml
ansible-vault encrypt group_vars/web/vault.yml --vault-password-file ~/.vault_pass
cat group_vars/web/vault.yml
# $ANSIBLE_VAULT;1.1;AES256...

# View
ansible-vault view group_vars/web/vault.yml --vault-password-file ~/.vault_pass

# Edit
ansible-vault edit group_vars/web/vault.yml --vault-password-file ~/.vault_pass

# Encrypt string
ansible-vault encrypt_string 'MySecret!' --name 'my_secret' --vault-password-file ~/.vault_pass

# --- inventory control local ---
cat > inventory-control.ini <<'EOF'
[control]
localhost ansible_connection=local ansible_python_interpreter=/usr/bin/python3.12

[control:vars]
app_port=8080
EOF

# --- playbook ---
cat > playbooks/test-vault.yml <<'EOF'
---
- hosts: control
  connection: local
  gather_facts: false
  tasks:
    - debug:
        var: db_user
    - debug:
        msg: "Password cargado OK"
      no_log: true
EOF

# --- Ejecuciones ---
# 1. Interactiva
ansible-playbook playbooks/test-vault.yml -i inventory-control.ini --ask-vault-pass

# 2. Con archivo
ansible-playbook playbooks/test-vault.yml -i inventory-control.ini --vault-password-file ~/.vault_pass

# 3. Con ansible.cfg (sin flag)
cat > ansible.cfg <<'EOF'
[defaults]
inventory = ./inventory-control.ini
vault_password_file = ~/.vault_pass
stdout_callback = default
interpreter_python = auto_silent
EOF
ansible-playbook playbooks/test-vault.yml

# 4. Con env var
ANSIBLE_VAULT_PASSWORD_FILE=~/.vault_pass ansible-playbook playbooks/test-vault.yml -i inventory-control.ini

# 5. Con script (para HashiCorp Vault)
cat > ~/vault_pass.sh <<'EOF'
#!/bin/bash
echo "telcel123"
EOF
chmod +x ~/vault_pass.sh
ansible-playbook playbooks/test-vault.yml -i inventory-control.ini --vault-password-file ~/vault_pass.sh
```

---

## 11. Matriz Decisión TELCEL - ¿Qué método usar?

| Ambiente | Método | Ejemplo |
|----------|--------|---------|
| Lab Sesión 8 local | `--ask-vault-pass` | Interactivo, aprendes |
| Dev con 1 password | `ansible.cfg` + `~/.vault_pass` | Cómodo, sin flag |
| CI/CD Jenkins | `--vault-password-file /etc/ansible/vault_pass` | Archivo en servidor CI |
| Prod con HCP Vault | `ANSIBLE_VAULT_PASSWORD_FILE=~/vault_pass.sh` | Script que llama a `vault kv get` |
| Multi-ambiente prod/lab | `--vault-id prod@... --vault-id lab@...` | 2 passwords distintos |

---

> Fin Documento Único Sesión 8 - Completo con Teoría + Ejemplos + Labs
> TELCEL - Rocky 10.1 - Ansible 2.16.16
