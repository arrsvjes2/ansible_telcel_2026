# Anexo Sesión 2 - Variables Mágicas


---

## 13. Variables Mágicas de Ansible - El Superpoder TELCEL

Las variables mágicas son variables que Ansible SIEMPRE te da, sin que las definas. No vienen de facts, vienen del motor. Te permiten que un playbook sepa quién es, dónde está y quiénes son sus vecinos.

### 13.1 Lista Clave (Ansible 2.14+)

```text
1. inventory_hostname        -> Nombre en inventory (servera) - NO es hostname del SO
2. inventory_hostname_short  -> Parte corta antes del punto
3. group_names               -> Lista grupos del host actual ['web','lab']
4. groups                    -> Dict con TODOS los grupos {"web": ["servera","serverb"], "db": ["serverd","servere"]}
5. hostvars                  -> Dict gigante con variables de TODOS los hosts
6. play_hosts / ansible_play_hosts -> Hosts del play actual
7. inventory_dir             -> Directorio del inventory (rutas relativas seguras)
8. inventory_file            -> Ruta completa al inventory
9. ansible_check_mode        -> Bool True si --check
10. ansible_version          -> {"full":"2.14.5", "major":2}
```

### 13.2 Caso de Uso #1 - hostvars (El más importante TELCEL)

**Problema real:** Tus 3 webs necesitan saber la IP del primario de DB sin hardcodear.

```yaml
# group_vars/web.yml
db_primary_host: "{{ groups['db'][0] }}"  # serverd

# templates/app.conf.j2
db_host={{ hostvars[db_primary_host]['ansible_facts']['default_ipv4']['address'] }}
# Resultado: db_host=10.10.1.14 - se actualiza sola si cambia IP

# Playbook demo
- name: Demo hostvars TELCEL
  hosts: web
  gather_facts: yes
  tasks:
    - debug:
        msg: "Soy {{ inventory_hostname }} y mi DB es {{ groups['db'][0] }} con IP {{ hostvars[groups['db'][0]]['ansible_facts']['default_ipv4']['address'] }}"
```

**Casos de uso hostvars:**
- Web necesita IP de DB primario
- Generar cluster config con lista de todos los nodos web
- Balanceador necesita lista de backends

### 13.3 Caso de Uso #2 - group_names

```yaml
- debug:
    msg: "Soy web con puerto {{ http_port }}"
  when: "'web' in group_names"

- template:
    src: web-lab.conf.j2
    dest: /etc/app.conf
  when: 
    - "'web' in group_names"
    - "'lab' in group_names"
```

### 13.4 Caso de Uso #3 - inventory_hostname vs ansible_hostname

```yaml
- debug:
    msg: |
      inventory_hostname={{ inventory_hostname }}  # servera (controlas tú)
      ansible_hostname={{ ansible_facts['hostname'] }}  # hostname real SO

# Buena práctica: Usa inventory_hostname para configs lógicas
```

### 13.5 Caso de Uso #4 - groups para /etc/hosts dinámico

```yaml
- name: Generar /etc/hosts TELCEL con variables mágicas
  hosts: all
  become: yes
  tasks:
    - blockinfile:
        path: /etc/hosts
        block: |
          # BEGIN ANSIBLE TELCEL LAB
          {% for host in groups['lab'] %}
          {{ hostvars[host]['ansible_facts']['default_ipv4']['address'] }} {{ host }} {{ hostvars[host]['inventory_hostname_short'] }}
          {% endfor %}
          # END ANSIBLE
        marker: "# {mark} TELCEL LAB"
```

### 13.6 Caso de Uso #5 - inventory_dir para rutas seguras

```yaml
# MALO - solo funciona en tu laptop
- copy:
    src: /home/men/ansible-telcel/files/motd.txt
    dest: /etc/motd

# BUENO - con mágica
- copy:
    src: "{{ inventory_dir }}/files/motd.txt"
    dest: /etc/motd
```

### 13.7 LAB 2.4 - Variables Mágicas (40 min)

```yaml
# playbooks/session2-magic-vars.yml
---
- name: LAB 2.4 - Variables Magicas TELCEL
  hosts: all
  gather_facts: yes
  tasks:
    - debug:
        msg: |
          inventory_hostname={{ inventory_hostname }}
          inventory_hostname_short={{ inventory_hostname_short }}
          group_names={{ group_names }}
          inventory_dir={{ inventory_dir }}
          inventory_file={{ inventory_file }}
          ansible_version={{ ansible_version.full }}

    - debug:
        var: groups

    - name: Mostrar DB primario via groups + hostvars
      debug:
        msg: "Soy web {{ inventory_hostname }}, mi DB es {{ groups['db'][0] }} con IP {{ hostvars[groups['db'][0]]['ansible_facts']['default_ipv4']['address'] | default('sin facts') }}"
      when: "'web' in group_names"

    - copy:
        content: |
          my_name={{ inventory_hostname }}
          my_groups={{ group_names | join(',') }}
          db_primary={{ groups['db'][0] }}
          db_ip={{ hostvars[groups['db'][0]]['ansible_facts']['default_ipv4']['address'] }}
          web_cluster={{ groups['web'] | join(',') }}
        dest: /tmp/magic-vars.conf
      when: "'web' in group_names"
```

```bash
ansible-playbook playbooks/session2-magic-vars.yml --check --diff --limit servera
ansible-playbook playbooks/session2-magic-vars.yml --diff
ansible web -a "cat /tmp/magic-vars.conf"
ansible all -a "cat /etc/hosts | grep -A10 TELCEL"
```

### 13.8 Tabla Resumen

| Variable Mágica | Tipo | Uso TELCEL |
|-----------------|------|------------|
| inventory_hostname | string | Nombre en inventory - configs lógicas |
| inventory_hostname_short | string | Nombre corto sin dominio |
| group_names | list | when: 'web' in group_names |
| groups | dict | groups['db'][0] primer DB |
| hostvars | dict | hostvars[groups['db'][0]]['ansible_facts']['default_ipv4']['address'] |
| inventory_dir | string | {{ inventory_dir }}/files/ rutas relativas |
| inventory_file | string | Debug qué inventory usas |
| ansible_check_mode | bool | when: not ansible_check_mode |
| ansible_version | dict | Version check |
| play_hosts | list | Hosts del play actual |

**Próxima: Sesión 3 - Tasks, Handlers y Notify**
