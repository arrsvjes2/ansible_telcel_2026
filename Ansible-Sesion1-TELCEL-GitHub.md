# Workshop / Taller Ansible Intermedio para TELCEL
**9 Sesiones x 3 Horas | 27 Horas Totales**  
**Modalidad Práctica - Laboratorio Podman**  
**Septiembre 2026 - Cliente: TELCEL**  
**HPE Advisory & Professional Services**

---

## 📚 Contenido

1. [Introducción y Diagrama de Laboratorio](#1-introducción)
2. [Pruebas Básicas de Conectividad](#2-pruebas-conectividad)
3. [Gestión de Llaves SSH](#3-gestion-llaves-ssh)
4. [Sesión 1: Fundamentos Sólidos y Buenas Prácticas](#4-sesión-1)
5. [Anexo: Uso de group_vars y host_vars](#5-anexo-group_vars)

---

## 1. Introducción

### Diagrama de laboratorio

```text
[control] 10.89.0.2  -> nodo control (ansible)
[servera] 10.89.0.3
[serverb] 10.89.0.4
[serverc] 10.89.0.5
[serverd] 10.89.0.6
[servere] 10.89.0.7

Red: podman network ansible_lab / 10.89.0.0/24
```

### Usuarios de sistema operativo

```text
ansible_servera_1 - ansible_servere_1
user: ansible
groups: ansible(1000), wheel (10)
password: linux123
sudo: NOPASSWD
```

---

## 2. Pruebas Básicas de Conectividad

Desde el shell del contenedor de control, conectarse por SSH con el usuario `ansible` y password `linux123` usando el nombre de contenedor a cada uno de los nodos gestionados. Primera conexión pide aceptar fingerprint.

```bash
[root@014737bdb0e9 playbooks]# ssh ansible@ansible_serverb_1
The authenticity of host 'ansible_serverb_1 (10.89.0.4)' can't be established.
ED25519 key fingerprint is SHA256:6M2SWTxz6zblXO1awRQLzbXw9a3vzI07IUb+WFkjhAI.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'ansible_serverb_1' (ED25519) to the list of known hosts.
ansible@ansible_serverb_1's password: linux123
[ansible@b8c62d58f3ab ~]$ exit
logout
Connection to ansible_serverb_1 closed.

# Repetir para cada nodo:
ssh ansible@ansible_serverc_1
ssh ansible@ansible_serverd_1
ssh ansible@ansible_servere_1
```

---

## 3. LAB 1: Ansible con Podman - Conectividad y Primer Playbook

### Objetivo

Levantar 1 nodo de control + 5 nodos administrados y lograr `ansible all -m ping`.

### Parte 1 - Infraestructura

Están en `~/ansible` con tu `compose.yaml` o `podman-compose.yaml`:

```bash
# 1. Levantar el lab
podman-compose up -d
# o
podman compose up -d

# Verificar
podman ps
# deben ver: control, servera, serverb, serverc, serverd, servere

# 2. Entrar al nodo de control
podman exec -it ansible_control_1 bash
# dentro ya eres ansible@control

pwd
# /home/ansible/playbooks

# Tu control-volume de tu host ya se ve adentro como /home/ansible/playbooks
```

### Parte 2 - Configuración base de Ansible

Dentro del contenedor de control:

```bash
mkdir -p /home/ansible/playbooks/test
cd /home/ansible/playbooks/test

# 1. Crea el ansible.cfg
cat > ansible.cfg <<'EOF'
[defaults]
remote_user = ansible
inventory = inventory
host_key_checking = False

[privilege_escalation]
become=false
EOF

cat ansible.cfg

# 2. Crea el inventario - OJO: en Podman los nombres son los del compose
cat > inventory <<'EOF'
[webservers]
servera.lab.example.com ansible_host=servera
serverb.lab.example.com ansible_host=serverb
serverc.lab.example.com ansible_host=serverc
serverd.lab.example.com ansible_host=serverd
servere.lab.example.com ansible_host=servere
EOF

# Nota: si usas podman-compose con prefijo ansible_, tus hosts son ansible_servera_1 etc.
# como tu tenias: ansible_host=ansible_servera_1
# Verificalo con:
getent hosts servera
ping servera

cat inventory
```

### Parte 3 - Prueba de conectividad

```bash
# Prueba con password - te pedirá la pass de ansible en cada nodo
ansible all -m ping -k

# Si falla, verifica:
# 1. que sshd esté corriendo en los nodos:
podman exec servera ps aux | grep sshd

# 2. que puedas hacer ping por nombre:
ping servera

# 3. que el usuario ansible exista en servera-e

# Si te responde pong en los 5, ya pasaste el 50% del lab.
```

### Parte 4 - Ya sin password (obligatorio para el playbook)

```bash
ssh-keygen -t ed25519 -C "ansible@control" -f ~/.ssh/id_ed25519 -N ""

for i in servera serverb serverc serverd servere; do
  ssh-copy-id ansible@$i
done

# Ahora ya sin -k
ansible all -m ping
```

### Parte 5 - Tu primer playbook

```yaml
cat > motd.yaml <<'EOF'
---
- name: Configurar MOTD del lab
  hosts: all
  tasks:
    - name: Crear mensaje de bienvenida
      ansible.builtin.copy:
        content: "Bienvenido al lab Podman - {{ inventory_hostname }} - HPE / Telcel
"
        dest: /etc/motd
EOF

# Validar sintaxis
ansible-playbook --syntax-check motd.yaml

# Dry-run
ansible-playbook motd.yaml --check

# Ejecución real
ansible-playbook motd.yaml

# Verifica
ansible all -a "cat /etc/motd"
```

---

## 4. Crear la llave de gestión SSH - Paso a paso detallado

### 1. Entrar al control y volverse usuario ansible

```bash
podman exec -it ansible_control_1 bash
# o si tu contenedor se llama control
podman exec -it control bash

whoami
# debe ser ansible, si eres root:
su - ansible
cd ~
pwd
# /home/ansible
```

### 2. Generar la llave SSH

> Opciones: `-t ed25519` es más rápido y moderno que rsa. `-N ""` es sin passphrase para que Ansible no pida nada.

```bash
ssh-keygen -t ed25519 -C "ansible@control" -f ~/.ssh/id_ed25519 -N ""

# Verifica que se creó
ls -l ~/.ssh/
# id_ed25519  -> privada
# id_ed25519.pub -> publica

cat ~/.ssh/id_ed25519.pub
# Si ya tienes una llave y te pregunta overwrite, pon n si no quieres reemplazarla.
# Para este lab si puedes reemplazar con y.
```

### 3. Copiar la llave a los 5 nodos

Tus nodos se llaman: `ansible_servera_1`, `ansible_serverb_1`, `ansible_serverc_1`, `ansible_serverd_1`, `ansible_servere_1`

#### Opción A - Manual (la que deben aprender primero)

```bash
# Te va a pedir password linux123 en cada uno:

ssh-copy-id ansible@ansible_servera_1
ssh-copy-id ansible@ansible_serverb_1
ssh-copy-id ansible@ansible_serverc_1
ssh-copy-id ansible@ansible_serverd_1
ssh-copy-id ansible@ansible_servere_1

# Cuando te pida:
# Are you sure you want to continue connecting (yes/no)? -> escribe yes
# password: -> escribe linux123
```

#### Opción B - Automática para todo el lab

```bash
# Instala sshpass en el control (Rocky):
sudo dnf install -y sshpass

# Copia en loop
for nodo in ansible_servera_1 ansible_serverb_1 ansible_serverc_1 ansible_serverd_1 ansible_servere_1; do
  echo ">>> Copiando a $nodo"
  sshpass -p 'linux123' ssh-copy-id -o StrictHostKeyChecking=no ansible@$nodo
done
```

### 4. Verificar que ya funciona sin password

```bash
# Prueba individual
ssh ansible@ansible_servera_1 "hostname; whoami"
ssh ansible@ansible_serverb_1 "hostname; whoami"

# Prueba masiva con ansible (ya sin -k)
cd /home/ansible/playbooks/test
ansible all -m ping

# Debe darte 5 SUCCESS así:
# ansible_servera_1 | SUCCESS => { "ping": "pong" }
```

### 5. Si algo falla - troubleshooting rápido

```bash
# 1. ¿El sshd está arriba en los nodos?
podman exec ansible_servera_1 ps aux | grep sshd

# 2. ¿El usuario ansible existe y tiene password linux123 en los nodos?
podman exec ansible_servera_1 id ansible
# Si no existe, en cada nodo:
useradd ansible
echo 'linux123' | passwd --stdin ansible

# 3. ¿Permisos de .ssh mal?
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_ed25519.pub
chmod 600 ~/.ssh/authorized_keys

# 4. Prueba verbose
ssh -vvv ansible@ansible_servera_1
```

```bash
# Con esto ya puedes quitar el -k de todos tus comandos:

# Antes
ansible all -m ping -k

# Ahora
ansible all -m ping
ansible all -a "uptime"
```

---

## 5. Sesión 1: Fundamentos Sólidos y Buenas Prácticas

**Cliente:** TELCEL | **Duración:** 3 Horas | **Fecha inicio:** 27 de Septiembre 2026  
**Modalidad:** Práctico con laboratorio Podman/Docker  
**Perfil:** SysAdmins, SREs, Admins Linux

### Objetivo de la Sesión

- Entender la arquitectura Control vs Managed y el modelo Push idempotente de Ansible
- Montar una estructura profesional de proyecto que evite deuda técnica
- Utilizar flags de ejecución segura `--check --diff --limit`
- Implementar linting automático con `ansible-lint` y `yamllint`
- Ejecutar el Lab 1: Laboratorio con podman-compose y playbook `day0-setup.yml`

### Agenda Detallada (3 Horas)

| Bloque | Tema | Tiempo |
|--------|------|--------|
| 1 | Arquitectura Control vs Managed, Push Model, Idempotencia | 30 min |
| 2 | Principio DRY, No Hardcodear, ansible-doc | 20 min |
| 3 | Estructura Profesional, ansible.cfg, Inventarios | 40 min |
| 4 | Ejecución Segura --check --diff --limit + Linting | 20 min |
| 5 | LAB 1 - Setup Podman + day0-setup.yml | 70 min |

### 1. Arquitectura Ansible

**Control Node:** Donde se ejecuta ansible. Requiere Python 3.9+, Ansible Core, colecciones, Vault. No necesita agente.  
**Managed Nodes:** Cualquier host con Python + SSH (Linux) o WinRM (Windows). Un control puede manejar 1000+ nodos.

**Ventajas del modelo Push (vs Pull Puppet/Chef):**
- No necesitas agente en los managed
- No abres puertos extra, aprovechas bastion SSH existente
- Conexión bajo demanda, ideal para Telcel con redes segmentadas

> **Concepto clave:** Idempotencia - Correr el playbook 1 o 100 veces deja el sistema en el mismo estado.

#### Ejemplo Idempotente vs No Idempotente

```yaml
# MAL - No idempotente:
- name: Mala práctica
  shell: echo "hola" >> /etc/motd
  # Cada corrida agrega línea

# BIEN - Idempotente:
- name: Buena práctica
  ansible.builtin.copy:
    dest: /etc/motd
    content: "hola"

# Clave SysAdmin: En la salida distinguir changed vs ok
```

### 2. Reglas de Oro TELCEL

**DRY - Don't Repeat Yourself:** Si copias/pegas 3 tasks iguales, es un role o un loop. Caso real: 200 servidores web con mismo playbook, solo cambian variables por datacenter.

**No Hardcodear:** IP, passwords, nombres NUNCA en tasks. Todo va a inventory o `group_vars/all.yml`

Usa `ansible-doc`: `ansible-doc package`, `ansible-doc copy`, `ansible-doc template`

### 3. Estructura Profesional de Proyecto

```text
ansible-telcel/
├── ansible.cfg
├── inventory/
│   ├── prod.ini
│   └── lab/hosts.ini
├── group_vars/
│   ├── all.yml
│   └── web.yml
├── host_vars/
├── roles/
├── collections/
├── playbooks/
│   └── day0-setup.yml
├── templates/
│   └── chrony.conf.j2
└── vault/
    └── vault.yml
```

**ansible.cfg - Config Base Telcel:**

```ini
[defaults]
inventory = ./inventory/lab/hosts.ini
roles_path = ./roles:./collections
forks = 20
host_key_checking = False
retry_files_enabled = False
stdout_callback = yaml

# forks = paralelismo, 20 ideal para 4 nodos lab, 50-100 en prod con bastion
# retry_files_enabled = False evita archivos .retry basura
```

**Inventario Lab:**

```ini
[web]
web01 ansible_host=10.0.1.11
web02 ansible_host=10.0.1.12

[db]
db01 ansible_host=10.0.1.21

[lab:children]
web
db

# Nunca pongas user/pass en inventory. Usa --user o group_vars
# Agrupa por rol, por ambiente, por datacenter para facilitar --limit
```

### 4. Modo Seguro: --check --diff --limit

```bash
# --check : Dry-run, no cambia nada
# --diff : Muestra diff exacto
# --limit : Apunta con francotirador

# Combo Telcel recomendado:
ansible-playbook playbooks/day0-setup.yml --check --diff --limit web
ansible-playbook playbooks/day0-setup.yml --diff
```

### 5. Linting

```bash
yamllint . 
# detecta indentación, truthy (yes/no vs true/false)

ansible-lint
# detecta uso de command vs package, no FQCN, etc.

# Integrar a Git con pre-commit hook
# Regla: Todo playbook debe pasar lint antes de PR
```

### LAB 1: Setup Completo

```bash
# Levantar lab
podman-compose up -d

# Setup SSH (dentro de control)
ssh-keygen -t ed25519 -f /root/.ssh/id_ed25519 -N ""

for h in 10.0.1.11 10.0.1.21 10.0.1.31 10.0.1.41; do
  ssh-copy-id -o StrictHostKeyChecking=no root@$h
done

# Validación
ansible all -m ping
ansible-inventory --graph

# Ejecución segura
ansible-playbook playbooks/day0-setup.yml --check --diff --limit web
ansible-playbook playbooks/day0-setup.yml --diff

# Validación final
ansible lab -a "cat /etc/motd"
ansible lab -a "timedatectl"
```

---

## 6. ANEXO - Sesión 1: Uso de group_vars y host_vars con Tareas Reales

### Concepto Clave - Precedencia

Ansible carga variables en orden de menor a mayor precedencia. La que carga al final gana:

1. `group_vars/all.yml` (menos prioridad - aplica a todos)
2. `group_vars/web.yml` o `group_vars/db.yml` (aplica a grupo)
3. `host_vars/web01.yml` (más prioridad - solo a ese host)

```text
group_vars/
├── all.yml          -> http_port: 80, app_name: generic
├── web.yml          -> http_port: 8080, app_name: portal-telcel-web, max_clients: 200
└── db.yml           -> http_port: 3306, app_name: telcel-db-primary

host_vars/
├── web01.yml        -> http_port: 8081, max_clients: 300, is_canary: true
└── db01.yml         -> http_port: 3307, max_connections: 1000
```

### Ejemplo 1 - group_vars/web.yml

```yaml
---
# Variables para TODOS los web servers
http_port: 8080
https_port: 8443
app_name: "portal-telcel-web"
max_clients: 200
motd_role: "WEB SERVER - Portal Telcel"

repos_extra:
  - nginx-stable
  - telcel-custom-web

nginx_worker_processes: 2
nginx_config:
  server_name: "{{ inventory_hostname }}.telcel.lab"
  listen_port: "{{ http_port }}"

web_packages:
  - nginx
  - php-fpm
```

### Ejemplo 2 - host_vars/web01.yml (sobrescribe)

```yaml
---
# Variables SOLO para web01 - mayor precedencia
http_port: 8081
max_clients: 300
motd_role: "WEB01 - PROD CANARY"

is_canary: true
canary_weight: 10

nginx_config:
  server_name: "web01.telcel.lab"
  listen_port: 8081
  custom_header: "X-Canary: true"

extra_vip: "10.0.1.100"
```

### Ejemplo 3 - Playbook demo-vars.yml

```yaml
---
- name: DEMO Variables Telcel - Precedencia y uso real
  hosts: lab
  gather_facts: false
  become: true
  tasks:
    - name: 1. Mostrar de donde viene cada variable
      ansible.builtin.debug:
        msg: |
          Host: {{ inventory_hostname }}
          app_name: {{ app_name }}
          http_port: {{ http_port }}
          motd_role: {{ motd_role }}

    - name: 2. Crear archivo de config usando group_vars
      ansible.builtin.template:
        src: app.conf.j2
        dest: "/tmp/{{ app_name }}.conf"
        mode: '0644'

    - name: 3. Tarea solo si max_clients > 250
      ansible.builtin.debug:
        msg: "ALERTA: {{ inventory_hostname }} alto trafico - max_clients={{ max_clients }}"
      when: max_clients is defined and max_clients | int > 250

    - name: 4. Configurar MOTD con host_vars
      ansible.builtin.copy:
        dest: /etc/motd
        content: |
          {{ motd_text }}
          App: {{ app_name }}
          Puerto: {{ http_port }}
          VIP: {{ extra_vip | default('N/A') }}
```

### Cómo ejecutarlo en clase

```bash
# 1. Ver precedencia real
ansible-playbook playbooks/demo-vars.yml --diff

# 2. Solo web
ansible-playbook playbooks/demo-vars.yml --limit web --tags motd --diff
ansible web -a "cat /etc/motd"

# 3. Ver archivos generados
ansible-playbook playbooks/demo-vars.yml --tags config
ansible web -a "cat /tmp/portal-telcel-web.conf"

# 4. Validar when
ansible-playbook playbooks/demo-vars.yml --tags high-traffic

# 5. Debug precedencia
ansible-inventory --host web01
ansible-inventory --host web02
```

### Ejercicio para alumnos (10 min)

1. Crea `host_vars/web02.yml` con `http_port: 8082` y `is_canary: false`
2. Crea `group_vars/app.yml` con `app_name: telcel-app-backend` y `http_port: 9000`
3. Ejecuta: `ansible-playbook playbooks/demo-vars.yml --limit web02 --diff`
4. Pregunta clave: ¿Qué pasa si defines misma variable en `all.yml`, `web.yml` y `web01.yml`? Respuesta: `host_vars/web01.yml` siempre gana.

---

> **Cliente:** TELCEL | **Fecha:** 27 Septiembre 2026 | **Instructor:** TC Senior HPE | **Lab:** Podman 1 Control + 5 Managed (ansible_servera_1 - ansible_servere_1) | **User:** ansible / linux123
