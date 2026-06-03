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
  to be loaded int `OpenBao-ca` variable.

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
  openbao_keys_output: "{{ openbao_config_path }}/unseal.json"
  ```

  To automatically initialise OpenBao, set `openbao_init` to true, and provide `openbao_key_shares` and
  `openbao_key_thershold` variables to specify the number of unseal keys to be generated.

  Initialisation will generate JSON file, `openbao_keys_output`, containing the unseal keys and root token

- OpenBao unseal and unseal service

  ```yml
  openbao_unseal: false
  openbao_unseal_service: false
  ```

  To automatically unseal OpenBao, set `openbao_unseal` to true. The unsealing process will use keys from
  `openbao_keys_output` file

  Systemd service can be created to automatically unseal OpenBao whenever OpenBao service is started or restarted. To
  enable it, set `openbao_unseal_service` to true. Oneshot `OpenBao-unseal`. This service also uses
  `openbao_keys_output` file.

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
      hcl: |
        path "secret/*" {
          capabilities = [ "create", "read", "update", "delete", "list", "patch" ]
        }
    - name: read
      hcl: |
        path "secret/*" {
          capabilities = [ "read" ]
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
  hosts: OpenBao-server
  become: true
  gather_facts: true
  vars:
    server_hostname: OpenBao.ricsanfre.com
    ssl_key_size: 4096
    key_type: RSA
    country_name: ES
    email_address: admin@ricsanfre.com
    organization_name: Ricsanfre
    ansible_user: root

  pre_tasks:
    - name: Generate custom CA
      include_tasks: tasks/generate_custom_ca.yml
      args:
        apply:
          delegate_to: localhost
          become: false
    - name: Generate customCA-signed SSL certificates for minio
      include_tasks: tasks/generate_ca_signed_cert.yml
      args:
        apply:
          delegate_to: localhost
          become: false

    - name: Load tls key and cert
      set_fact:
        openbao_key: "{{ lookup('file', 'certificates/' + server_hostname + '.key') }}"
        openbao_cert: "{{ lookup('file', 'certificates/' + server_hostname + '.pem') }}"
        openbao_ca: "{{ lookup('file', 'certificates/CA.pem') }}"

  roles:
    - role: ricsanfre.OpenBao
      openbao_enable_tls: true
      custom_ca: true
      openbao_init: true
      openbao_unseal: true
      openbao_unseal_service: true
      tls_skip_verify: true
      display_init_response: true
      # Configure KV
      openbao_kv_secrets:
        path: secret

      # Policies
      policies:
        - name: write
          hcl: |
            path "secret/*" {
              capabilities = [ "create", "read", "update", "delete", "list", "patch" ]
            }
        - name: read
          hcl: |
            path "secret/*" {
              capabilities = [ "read" ]
            }
```

`pre-tasks` section include tasks to generate a custom CA, and OpenBao's private key and certificate and load them into
`openbao_key`, `openbao_cert` and `OpenBao-ca` variables.

Where `generate_custom_ca.yml` contain the tasks for generating a custom CA:

```yml
---
- name: Create CA key
  openssl_privatekey:
    path: certificates/CA.key
    size: "{{ ssl_key_size | int }}"
    mode: 0644
  register: ca_key

- name: create the CA CSR
  openssl_csr:
    privatekey_path: certificates/CA.key
    common_name: Ricsanfre CA
    use_common_name_for_san: false  # since we do not specify SANs, don't use CN as a SAN
    basic_constraints:
      - 'CA:TRUE'
    basic_constraints_critical: true
    key_usage:
      - keyCertSign
    key_usage_critical: true
    path: certificates/CA.csr
  register: ca_csr

- name: sign the CA CSR
  openssl_certificate:
    path: certificates/CA.pem
    csr_path: certificates/CA.csr
    privatekey_path: certificates/CA.key
    provider: selfsigned
  register: ca_crt


```

And `generate_ca_signed_certificate.yml` contain the tasks for generating OpenBao's key and certificate signed by custom
CA:

```yml
---
- name: Create private key
  openssl_privatekey:
    path: "certificates/{{ server_hostname }}.key"
    size: "{{ ssl_key_size | int }}"
    type: "{{ key_type }}"
    mode: 0644

- name: Create CSR
  openssl_csr:
    path: "certificates/{{ server_hostname }}.csr"
    privatekey_path: "certificates/{{ server_hostname }}.key"
    country_name: "{{ country_name }}"
    organization_name: "{{ organization_name }}"
    email_address: "{{ email_address }}"
    common_name: "{{ server_hostname }}"
    subject_alt_name: "DNS:{{ server_hostname }},IP:{{ ansible_facts['default_ipv4']['address'] }},IP:127.0.0.1"

- name: CA signed CSR
  openssl_certificate:
    csr_path: "certificates/{{ server_hostname }}.csr"
    path: "certificates/{{ server_hostname }}.pem"
    provider: ownca
    ownca_path: certificates/CA.pem
    ownca_privatekey_path: certificates/CA.key
```

## License

MIT

## Author Information

Created by Olaf Lipinski, from the role by Ricardo Sanchez (ricsanfre)
