# Ansible Role: OpenBao Server Installation and Configuration

This role installs and configures [OpenBao](https://openbao.org) on a Linux server.

## Requirements

None

## Role Variables

Available variables are listed below along with default values (see `defaults\main.yaml`)

- OpenBao server installation details

  OpenBao UNIX user/group

  ```yml
  openbao_group: openbao
  openbao_user: openbao
  ```

  OpenBao package and version to be installed

  ```yml
  openbao_version: 2.5.4
  ```

  OpenBao installation Paths

  ```yml
  openbao_bin_path: /usr/local/bin
  openbao_config_path: /etc/openbao
  openbao_tls_path: /etc/openbao/tls
  openbao_plugin_path: /usr/local/lib/openbao/plugins
  openbao_data_path: /var/lib/openbao
  openbao_log_path: /var/log/openbao
  ```

- OpenBao TLS configuration

  ```yml
  openbao_enable_tls: false
  openbao_key: ""
  openbao_cert: ""
  custom_ca: false
  openbao_ca: ""
  # OpenBao service dns
  openbao_dns: ""
  ```

  To enable configuration of TLS set `openbao_enable_tls` to true and provide the private key and public certificate as
  content loaded into `openbao_key` and `openbao_cert` variables.

  if custom CA has been used to sign the TLS certificates, `custom_ca` need to be set to true, and CA certificate need
  to be loaded int `openbao-ca` variable.

  Set `openbao_dns` to FQDN of OpenBao service used to issue the certificate.

  They can be loaded from files using an ansible task like:

  ```yml
  - name: Load tls key and cert from files
    set_fact:
      openbao_key: "{{ lookup('file','certificates/{{ inventory_hostname }}_private.key') }}"
      openbao_cert: "{{ lookup('file','certificates/{{ inventory_hostname }}_public.crt') }}"
      openbao_ca: "{{ lookup('file','certificates/ca.crt') }}"
  ```

- OpenBao initialisation

  ```yml
  openbao_init: false
  openbao_key_shares: 1
  openbao_key_threshold: 1
  openbao_save_keys_to_file: false
  openbao_keys_output: "{{ openbao_config_path }}/unseal.json"
  ```

  To automatically initialise OpenBao, set `openbao_init` to true, and provide `openbao_key_shares` and
  `openbao_key_thershold` variables to specify the number of unseal keys to be generated.

  > [!CAUTION]
  > Initialisation can also output the keys to a file as per below. Use only for testing.

  When `openbao_save_keys_to_file` is set, initialisation will generate a JSON file containing the unseal keys and root
  token, and write them to `openbao_keys_output`. This file is a pre-requisite for the unseal service.

- OpenBao unseal service

  > [!CAUTION]
  > This is completely insecure, and NOT recommended for production setups. This option stores the root token as a text
  > file. Use only for testing.

  ```yml
  openbao_unseal: false
  openbao_unseal_service: false
  ```

  To automatically unseal OpenBao, set `openbao_unseal` to true. The unsealing process will use keys from
  `openbao_keys_output` file.

  Systemd service can be created to automatically unseal OpenBao whenever OpenBao service is started or restarted. To
  enable it, set `openbao_unseal_service` to true.

- KV secrets engine

  KV version 2 secret engine can be automatically enabled providing the following variables

  ```yml
  openbao_kv_secrets:
    path: secret
  ```

  KV version 2 will be enabled at path `secret`

- Policies

  ACL policies can be automatically configured, providing a name and the hcl content.

  ```yml
  policies:
  - name: write
    json: |
      {
        "path":{
          "secret/*":{
            "capabilities":[
              "create",
              "read",
              "update",
              "delete",
              "list",
              "patch"
            ]
          }
        }
      }
  - name: read
    json: |
      {
        "path":{
          "secret/*":{
            "capabilities":[
              "read"
            ]
          }
        }
      }
  ```

## Dependencies

None

## Example Playbook

The following playbook install and configure OpenBao, enabling TLS and generating custom CA signed SSL certificates. It
initialises and unseals OpenBao. It also enables KV version 2 at `secret` path and create a couple of policies (`read`
and `write`)

```yml
---
- name: Install and configure OpenBao Server
  hosts: openbao-server
  become: true
  gather_facts: true
  vars:
    server_hostname: openbao.example.com
    ssl_key_size: 4096
    key_type: RSA
    country_name: ES
    email_address: admin@example.com
    organization_name: Example
    ansible_user: root

  pre_tasks:
    - name: Load tls key and cert
      set_fact:
        openbao_key: "{{ lookup('file', 'certificates/' + server_hostname + '.key') }}"
        openbao_cert: "{{ lookup('file', 'certificates/' + server_hostname + '.pem') }}"
        openbao_ca: "{{ lookup('file', 'certificates/CA.pem') }}"

  roles:
    - role: olipinski.openbao
      openbao_enable_tls: true
      custom_ca: true
      openbao_init: true
      openbao_unseal: true
      openbao_unseal_service: true
      openbao_tls_skip_verify: true
      openbao_display_init_response: true
      # Configure KV
      openbao_kv_secrets:
        path: secret

      # Policies
      policies:
       - name: write
         json: |
           {
            "path":{
              "secret/*":{
                "capabilities":[
                  "create",
                  "read",
                  "update",
                  "delete",
                  "list",
                  "patch"
                ]
              }
            }
          }
       - name: read
         json: |
           {
            "path":{
              "secret/*":{
                "capabilities":[
                  "read"
                ]
              }
            }
          }
```

## License

MIT

## Author Information

Created by Olaf Lipinski, from the role by Ricardo Sanchez (ricsanfre)
