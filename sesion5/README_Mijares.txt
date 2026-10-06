# TALLER ANSIBLE TELCEL - SESION 5 - PAQUETE COMPLETO PARA MIJARES
HPE A&PS | 100% Offline Container Friendly | Sin Internet Sin Init

## Contenido:

### Sesion 5 Instructor Mijares:
- ansible-sesion5/SESION_5_Roles_Vault_Collections_TELCEL.docx
- ansible-sesion5/PLAN_INSTRUCTOR_SESION_5_Mijares.docx - Plan 3 horas

### Sesion 3 actualizada offline:
- ansible-sesion3/SESION_3_Tasks_Handlers_TELCEL.docx
- docs/SESION_3_Tasks_Handlers_TELCEL.md

### Sesion 5 markdown:
- docs/SESION_5_Roles_Vault_Collections_TELCEL.md
- docs/PLAN_INSTRUCTOR_SESION_5_Mijares.md

### Roles listos:
- docs/roles/telcel_app/ (tasks, handlers, defaults, templates)
- docs/roles/telcel_logrotate/

### Templates:
- docs/templates/*.j2 (todos offline con grep validate)

### Proyecto:
- docs/site.yml
- docs/ansible.cfg

## Instalacion en control:

cd ~/taller-ansible-telcel
unzip ... -d /tmp/paquete
cp -r /tmp/paquete/docs/roles/* roles/
cp /tmp/paquete/docs/site.yml .
cp /tmp/paquete/docs/ansible.cfg .
echo "TelcelVault2025" > ~/.vault_pass && chmod 600 ~/.vault_pass
ansible-vault create --vault-password-file ~/.vault_pass inventory/group_vars/web/vault.yml
ansible-playbook site.yml --vault-password-file ~/.vault_pass --diff

Para Mijares: Agenda 180 min en PLAN_INSTRUCTOR
