# hashicorp-vault-configuration

Ansible-роль идемпотентно настраивает в уже установленном и распечатанном HashiCorp Vault:

- аутентификацию LDAP для Microsoft Active Directory;
- список отсутствующих хранилищ KV v2;
- отдельные политики чтения/записи и только чтения для каждого KV;
- привязку политик к группам AD через LDAP group mapping.

Роль работает напрямую с HTTP API и не требует коллекции `community.hashi_vault`. Административный токен и LDAP bind password следует хранить в Ansible Vault либо получать lookup-плагином, а не записывать открытым текстом в репозиторий. Все API-задачи по умолчанию скрыты через `no_log`.

## Требования

- Ansible Core 2.13 или новее;
- инициализированный и распечатанный Vault;
- токен с правами на `sys/auth`, `sys/mounts`, `sys/policies/acl` и `auth/<ldap_path>`;
- сетевой доступ от `hashicorp_vault_configuration_delegate_to` к API Vault.

## Переменные

| Переменная | По умолчанию | Назначение |
|---|---|---|
| `hashicorp_vault_configuration_address` | `http://127.0.0.1:8200` | Адрес API Vault. |
| `hashicorp_vault_configuration_token` | `""` | Административный токен; обязателен. |
| `hashicorp_vault_configuration_hashi_vault_path_token` | `""` | Документированный путь для варианта с lookup HashiCorp Vault. |
| `hashicorp_vault_configuration_validate_certs` | `true` | Проверять TLS-сертификат API. |
| `hashicorp_vault_configuration_ca_path` | `""` | CA-файл для API Vault. |
| `hashicorp_vault_configuration_namespace` | `""` | Vault Enterprise namespace. |
| `hashicorp_vault_configuration_api_timeout` | `30` | Таймаут API в секундах. |
| `hashicorp_vault_configuration_delegate_to` | текущий host | Узел, с которого выполняются API-запросы. |
| `hashicorp_vault_configuration_no_log` | `true` | Скрывать чувствительные результаты задач. |
| `hashicorp_vault_configuration_ldap_path` | `ldap` | Mount path метода LDAP. |
| `hashicorp_vault_configuration_ldap_description` | описание AD | Описание auth mount. |
| `hashicorp_vault_configuration_ldap` | пример AD | Полное тело LDAP-конфигурации: URL, DN, фильтры, TLS и таймауты. |
| `hashicorp_vault_configuration_ldap_force_bindpass_update` | `false` | Явно отметить ротацию скрытого Vault параметра `bindpass` как изменение. |
| `hashicorp_vault_configuration_kv_v2` | `[]` | Список KV v2 и пар AD-групп. |
| `hashicorp_vault_configuration_policy_prefix` | `ad-kv` | Префикс автоматически создаваемых политик. |
| `hashicorp_vault_configuration_preserve_group_policies` | `true` | Сохранять сторонние политики в существующих LDAP mappings. |
| `hashicorp_vault_configuration_clean_confirm` | `false` | Защита запуска cleanup. |
| `hashicorp_vault_configuration_clean_disable_ldap` | `false` | При cleanup удалить LDAP auth mount. |
| `hashicorp_vault_configuration_clean_remove_mounts` | `false` | При cleanup удалить KV вместе со всеми данными. |

Каждый элемент `hashicorp_vault_configuration_kv_v2` принимает обязательные `mount`, `rw_group`, `ro_group` и необязательные `description`, `rw_policy`, `ro_policy`. Mount должен состоять из букв, цифр, `_` и `-`.

## Пример inventory

```yaml
all:
  children:
    vault:
      hosts:
        vault01.example.org:
      vars:
        hashicorp_vault_configuration_address: https://vault.example.org:8200
        hashicorp_vault_configuration_ca_path: /etc/pki/ca-trust/source/anchors/company-ca.pem
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
            description: Application secrets
            rw_group: AD-Vault-Applications-RW
            ro_group: AD-Vault-Applications-RO
          - mount: infrastructure
            rw_group: AD-Vault-Infrastructure-RW
            ro_group: AD-Vault-Infrastructure-RO
```

## Пример playbook

```yaml
---
- name: Configure Vault
  hosts: vault
  gather_facts: false
  roles:
    - role: hashicorp-vault-configuration
```

При повторном запуске роль обновляет LDAP-настройки и политики, добавляет отсутствующие KV v2 и не пересоздаёт существующие совместимые mount points. Если существующий mount имеет другой тип или KV v1, роль останавливается, чтобы не уничтожить данные.

## Теги

- `pre-req` — проверка переменных и готовности API;
- `user`, `install` — совместимые no-op этапы: роль не управляет ОС;
- `ssl` — проверка локального CA-файла;
- `config` — LDAP, KV, политики и группы;
- `clean` — удаление политик и, только при дополнительных флагах, LDAP/KV; всегда используется вместе с `never`.

## Molecule

Сценарий `default` поднимает systemd-контейнер RedOS, устанавливает и инициализирует Vault существующей ролью, применяет эту роль дважды в фазе idempotence, затем проверяет LDAP config, KV v2, ACL policies, mappings AD-групп и запись/чтение тестового секрета.

> Удаление KV mount необратимо удаляет его данные. Cleanup по умолчанию этого не делает.
