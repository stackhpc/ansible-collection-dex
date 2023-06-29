This role deploys Dex in standalone-HA configuration, using MySQL as the backend storage 

### Role variables

#### OCI image
##### Optional
* **dex_docker_name:** Name of the Dex docker container (default: "dex")
* **dex_docker_image:** Docker image to use (default: "quay.io/dex/dex")
* **dex_docker_tag:** Image tag to use (default: "v2.36.0")
* **dex_container.etc_hosts:** Dict of host-to-IP mappings (default: {})
 
#### Dex database configuration
##### Mandatory
* **dex_db_password:** Database password
* **dex_db_url_host:** Database host
* **dex_db_login_username:** Database user used to create Dex database (if `dex_db_ensure_database: true`)
* **dex_db_login_password:** Password for database user used to create Dex database (if `dex_db_ensure_database: true`)

##### Optional
* **dex_db_type:** Database type (only mysql tested, default: `"mysql"`)
* **dex_db_ensure_database:** Ensure that the Dex database and user exists (default: `true`, requires
  `dex_db_login_username` and `dex_db_login_password`)
* **dex_db_mysql_user_priv:** MySQL privileges string to set when ensuring Dex database user (default: `"'{{ dex_db_url_database }}.*:ALL'"`)
* **dex_db_mysql_user_host:** The "host" part of the MySQL username when ensuring Dex database user
  (default: `"%"`, all hosts)
* **dex_db_username:** Database username (default: `"dex"`)
* **dex_db_url_database:** Database name (default: `"dex"`)
* **dex_db_url_port:** Database port (default: `3306`)
* **dex_db_ssl:** Database uses TLS (default: `false`)

#### Dex configuration
##### Optional
* **dex_http_bind_interface:** Interface to bind the Dex http server (default: `"lo"`)
* **dex_http_port:** Port to bind the dex http server (default: `5556`)
* **dex_host_config_dir:** Host directory for Dex configuration (default: `"/etc/dex"`)
* **dex_config_file:** Dex config filename(default: `"config.yaml"`)
* **dex_issuer_tls:** Dex issuer listens on https (default: `false`)
* **dex_issuer_host:** Dex issuer hostname (default: `{{ dex_http_host }}`)
* **dex_issuer_port:** Dex issuer port (default: `{{ dex_http_port }}`)
* **dex_issuer_path:** Dex issuer path (default: `"/dex"`)
* **dex_expiry_devicerequests:** Dex devicerequest expiry (default: `"5m"`)
* **dex_expiry_signingkeys:** Dex signingkey expiry (default: `"6h"`)
* **dex_expiry_idtokens:** Dex idtoken expiry (default: `"24h"`)
* **dex_expiry_authrequests:** Dex authrequest expiry (default: `"24h"`)
* **dex_config_staticclients:** Dex staticClient config (default: `[]`)
* **dex_config_connectors:** Dex connectors config (default: `[]`)
* **dex_config_oauth2:** Dex oauth2 config (default: `{}`)

#### Dex container command
##### Optional
* **dex_docker_command:** Dex Docker command (default: `"dex serve /etc/dex/cfg/{{ dex_config_file }}"`)

##### Optional
* **dex_volumes_extra:** A list of extra volumes to be specified (default: `[]`)
* **dex_env_vars_overrides:** A dict of environment variables to set in the Dex container (default: `{}`)


Example playbook (used with OpenStack Kayobe)

```YAML
---
- name: Run Dex Install role
  vars:
    kolla_config_path: "{{ lookup('ansible.builtin.env', 'KOLLA_CONFIG_PATH') }}"
    kolla_passwords: "{{ lookup('ansible.builtin.unvault', kolla_config_path ~ '/passwords.yml') | from_yaml }}"
    # Kolla-ansible-deployed MariaDB credentials
    dex_db_login_password: "{{ kolla_passwords['database_password'] }}"
    dex_db_login_username: root
  hosts: controllers
  tasks:
    - name: Set a fact about the virtualenv on the remote system
      set_fact:
        virtualenv: "{{ ansible_python_interpreter | dirname | dirname }}"
      when:
        - ansible_python_interpreter is defined
        - not ansible_python_interpreter.startswith('/bin/')
        - not ansible_python_interpreter.startswith('/usr/bin/')
    
    - name: Ensure Python PyMySQL module is installed
      pip:
        name: pymysql
        extra_args: "{% if pip_upper_constraints_file %}-c {{ pip_upper_constraints_file }}{% endif %}"
        virtualenv: "{{ virtualenv is defined | ternary(virtualenv, omit) }}"
      become: "{{ virtualenv is not defined }}"
    
    - import_role: 
        name: stackhpc.dex.deploy
      vars:
        dex_config_staticclients:
          - id: openstack-keystone
            redirectURIs:
              - https://{{ public_net_name | net_vip_address }}:5000/redirect_uri
            name: OpenStack
            secret: "{{ dex_keystone_client_secret }}"
        dex_config_connectors:
          - type: microsoft
            id: microsoft
            name: Microsoft
            config:
              clientID: 
              clientSecret:
              redirectURI: "https://{{ dex_issuer_host }}{{ dex_issuer_path }}/callback"
              tenant: common
              scopes:
                - openid
                - email
                - groups
              emailToLowercase: true      
        dex_db_url_host: "{{ internal_net_name | net_vip_address }}"
        dex_db_password: "{{ dex_database_password }}"
        dex_http_bind_interface: "{{ internal_interface }}"
        dex_issuer_tls: true
        dex_issuer_host: identity.example.com
        dex_issuer_path: "/dex"
        dex_config_oauth2:
          responseTypes:
            - id_token
```
