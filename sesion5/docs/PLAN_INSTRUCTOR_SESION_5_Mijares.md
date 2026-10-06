
# Plan de Instructor - Sesión 5 - Mijares
## Roles, Collections y Vault | TELCEL | HPE A&PS | 3 Horas | Offline

### 1. Resumen
Convertir LABs Sesión 3 a Roles, meter Vault, site.yml. 100% offline sin internet ni systemd.

### 2. Checklist Pre-clase (Mijares)
- control af76376983e0 ansible 2.16.16
- ansible-lint via pipx
- 4 containers sin internet (a propósito)
- inventory/hosts OK
- Roles de respaldo en /mnt/data/docs/roles/
- ~/.vault_pass creado

### 3. Agenda 180 min
- 00:00-00:15 Recap + motivación Roles
- 00:15-00:35 Teoría Roles, galaxy --offline, ansible.cfg
- 00:35-01:20 LAB 5.1 telcel_app (45 min)
- 01:20-01:50 LAB 5.2 telcel_logrotate (30 min)
- 01:50-02:20 LAB 5.3 Vault (30 min)
- 02:20-02:50 LAB 5.4 site.yml (30 min)
- 02:50-03:00 Reto + cierre (10 min)

### 4. Guión

**LAB 5.1:** "Playbook gigante no escala, role sí. Defaults vs vars. Validate con grep porque no hay binario httpd. Sin dnf porque no hay internet."

**LAB 5.2:** "Sin systemctl, solo echo. Concepto handler."

**LAB 5.3:** "Secrets no van a git en claro. Vault + mode 0600 + no_log:true. Sin pass falla, con pass OK."

**LAB 5.4:** "site.yml orquesta todo. 2da corrida 0 changed = idempotencia."

### 5. Comandos

```bash
ansible-galaxy role init roles/telcel_app --offline
ansible-playbook playbooks/test-telcel-app.yml --check --diff --limit servera
echo "TelcelVault2025" > ~/.vault_pass && chmod 600 ~/.vault_pass
ansible-vault create --vault-password-file ~/.vault_pass inventory/group_vars/web/vault.yml
ansible-playbook site.yml --vault-password-file ~/.vault_pass --diff
ansible-playbook site.yml --vault-password-file ~/.vault_pass --diff # 2da 0 changed
```

### 6. Troubleshooting

- dnf falla -> Esperado offline, no usar dnf
- httpd -t no such file -> Usar grep -q Listen
- service falla systemd -> Usar command cat + debug
- vault pide pass interactivo -> Usar --vault-password-file
- 2da corrida changed -> Quitar ansible_date_time de template

### 7. Materiales

Entregar estructura vacía roles/, inventory/hosts, ansible.cfg con vault_password_file. No entregar vault en claro.

### 8. Cierre Checklist

Roles funcionando, vault cifrado, site.yml 0 changed 2da corrida, lint OK, zip entregable.

**Instructor:** Mijares | **Cliente:** TELCEL | **Fecha:** Sesión 5
