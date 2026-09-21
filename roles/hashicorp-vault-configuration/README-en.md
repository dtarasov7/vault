# hashicorp-vault-configuration

This Ansible role idempotently configures Microsoft Active Directory LDAP authentication, missing KV v2 mounts, per-mount read-write/read-only policies, and LDAP AD group mappings in an initialized and unsealed HashiCorp Vault.

It uses the Vault HTTP API and requires Ansible Core 2.13 or newer. Keep the administrative token and LDAP bind password in Ansible Vault or supply them through a lookup. API tasks use `no_log` by default.

## Main variables

| Variable | Default | Purpose |
|---|---|---|
| `hashicorp_vault_configuration_address` | `http://127.0.0.1:8200` | Vault API address. |
| `hashicorp_vault_configuration_token` | `""` | Required administrative token. |
| `hashicorp_vault_configuration_validate_certs` | `true` | Validate the API TLS certificate. |
| `hashicorp_vault_configuration_ca_path` | `""` | Optional Vault API CA file. |
| `hashicorp_vault_configuration_namespace` | `""` | Optional Enterprise namespace. |
| `hashicorp_vault_configuration_api_timeout` | `30` | API timeout in seconds. |
| `hashicorp_vault_configuration_delegate_to` | current host | Host issuing API requests. |
| `hashicorp_vault_configuration_no_log` | `true` | Hide sensitive task data. |
| `hashicorp_vault_configuration_ldap_path` | `ldap` | LDAP auth mount path. |
| `hashicorp_vault_configuration_ldap_description` | AD description | Auth mount description. |
| `hashicorp_vault_configuration_ldap` | AD example | Complete LDAP API configuration mapping. |
| `hashicorp_vault_configuration_ldap_force_bindpass_update` | `false` | Report a bind-password-only rotation as changed. |
| `hashicorp_vault_configuration_kv_v2` | `[]` | KV v2 definitions with `mount`, `rw_group`, and `ro_group`; optional `description`, `rw_policy`, `ro_policy`. |
| `hashicorp_vault_configuration_policy_prefix` | `ad-kv` | Generated policy name prefix. |
| `hashicorp_vault_configuration_preserve_group_policies` | `true` | Preserve unrelated policies on existing group mappings. |
| `hashicorp_vault_configuration_clean_confirm` | `false` | Cleanup safety switch. |
| `hashicorp_vault_configuration_clean_disable_ldap` | `false` | Disable LDAP during cleanup. |
| `hashicorp_vault_configuration_clean_remove_mounts` | `false` | Delete KV mounts and their data during cleanup. |

## Inventory and playbook

```yaml
hashicorp_vault_configuration_address: https://vault.example.org:8200
hashicorp_vault_configuration_token: "{{ vault_admin_token }}"
hashicorp_vault_configuration_ldap:
  url: ldaps://dc01.example.org:636
  binddn: CN=vault-bind,OU=Service Accounts,DC=example,DC=org
  bindpass: "{{ vault_ldap_bind_password }}"
  userdn: OU=Users,DC=example,DC=org
  userattr: sAMAccountName
  userfilter: "({{ '{{.UserAttr}}' }}={{ '{{.Username}}' }})"
  groupdn: OU=Groups,DC=example,DC=org
  groupattr: cn
  groupfilter: "(&(objectClass=group)(member:1.2.840.113556.1.4.1941:={{ '{{.UserDN}}' }}))"
  discoverdn: false
  deny_null_bind: true
  case_sensitive_names: false
  use_token_groups: false
  starttls: false
  insecure_tls: false
  tls_min_version: tls12
  tls_max_version: tls13
  certificate: ""
  connection_timeout: 30
  request_timeout: 90
hashicorp_vault_configuration_kv_v2:
  - mount: applications
    rw_group: AD-Vault-Applications-RW
    ro_group: AD-Vault-Applications-RO
```

```yaml
---
- name: Configure Vault
  hosts: vault
  gather_facts: false
  roles:
    - role: hashicorp-vault-configuration
```

Tags are `pre-req`, `user`, `install`, `ssl`, `config`, and opt-in `never,clean`. The Molecule `default` scenario provisions Vault, checks idempotence, LDAP, mounts, policies, group mappings, and an actual KV write/read. Existing incompatible mount types fail safely. KV deletion is disabled by default because it destroys data.
