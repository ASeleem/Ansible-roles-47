# Ansible roles

* Role to manage users on linux.  
* Role to install/configure/hardening Tomcat


# Users Role
Role to manage users on linux.

## Distros tested

* Ubuntu 18.04
* Debian 10.x
* CentOS / RHEL: 7.x

## Dependencies

Requires Ansible 2.6 or higher.

## ansible-vault

Use ansible-vault to encrypt sensitive info from git.

```bash
cat vars/secret
#encrypt if cleartext (before git commit/push)
ansible-vault encrypt vars/secret

#Edit encrypted file:
ansible-vault edit vars/secret

vi .vaultpass
-Enter the password for Ansible Vault from Password Safe
chmod 600 .vaultpass
vi ansible.cfg
#Insert the following lines
[defaults]
vault_password_file = ./.vaultpass
```

## .gitignore

```bash
vi .gitignore
#Insert the following lines
.vaultpass
.retry
secret
*.secret
```

## Default Settings for

```yaml
---
debug_enabled_default: true
default_update_password: on_create
default_shell: /bin/bash
```

## User Settings

File Location: vars/secret

* **username**: username - no spaces **(required)**
* **uid**: The numerical value of the user's ID (optional)
* **user_state**: present|lock **(required)**
* **password**: (will be generated randomly if not defined) sha512 encrypted password (optional). If not set, password is set to "!"
* **update_password**: always|on_create (optional, default is on_create to be safe).  
  **WARNING**: when 'always', password will be change to password value.  
  If you are using 'always' on an **existing** users, **make sure to have the password set**.
* **comment**: Full name and Department or description of application (optional) (But you should set this!)
* **primarygroup**: Primary group name (optional).
* **primarygid**: Primary group ID (optional). If same gid is reused on server the playbook will fail. If same duplicate group is specified with different gid, last configured will be used.
  **WARNING**: changing the primarygroup and/or primarygid of **existing** users will not change permissions of existing files belonging to that user. Also old entries will remain in /etc/group. Use with caution.
* **groups**: Comma separated list of groups the user will be added to (appended). If group doesn't exist it will be created on the specific server. This is not the primary group (primary group is not modified)
* **shell**: path to shell (optional, default is /bin/bash)
* **use_sudo**: yes|no (optional, default no)
* **use_sudo_nopass**: yes|no (optional, default no). yes = passwordless sudo.
* **system**: yes|no (optional, default no). yes = create system account (uid < 1000). Does not work on existing users.
* **servers**: sub-element list of servers where changes are made. **(required)**  
  These are the Ansible groups from your Ansible inventory file. In below examples, `webserver` would be the 3 servers in the `webserver` Ansible inventory `webserver1`, `webserver2`, and `webserver3`.  

Note:
  You can have duplicate usernames on different servers, if you want to have different settings. See below example of testuser102 has sudo on servers defined as the `webserver` group in the inventory, but no sudo on the `database` group.

## Example Ansible Inventory file

```yaml
[webserver]
webserver1
webserver2
webserver3

[database]
db1
db2
db3

[monitoring]
monitor1
```

## Example config file (vars/secret)

```yaml
---
users:
  - username: test01
    update_password: on_create
    comment: Test User 01
    shell: /bin/bash
    use_sudo: no
    use_sudo_nopass: no
    user_state: present
    servers:
      - webserver
      - database
      - monitoring

  - username: test02
    password: $6$F/KXFzMa$ZIDqtYtM6sOC3UmRntVsTcy1rnsvw.6tBquOhX7Sb26jxskXpve8l6DYsQyI1FT8N5I5cL0YkzW7bLbSCMtUw1
    update_password: always
    comment: Test User 02
    groups: testcommon, testgroup02web
    shell: /bin/sh
    use_sudo: yes
    user_state: present
    servers:
      - webserver

  - username: test03
    update_password: always
    comment: Test User 03
    groups: testcommon, testgroup03db
    shell: /bin/sh
    user_state: present
    servers:
      - database

  - username: test04
    user_state: present
    servers:
      - webserver

  - username: test04
    primarygroup: testgroup04primary
    use_sudo: no
    user_state: present
    servers:
      - webserver
      - monitoring

```

## Example Playbook create-users.yml

```bash
---
- hosts: '{{inventory}}'
  vars_files:
    - vars/secret
  become: yes
  roles:
  - create-users
```

## Prep

* Install Ansible
* Run Ansible Commands

## Usage

Create all users

```bash
ansible-playbook users.yaml --ask-vault-pass -i hosts.ini
```




# Tomcat Role
Ansible role to install and configure Apache Tomcat.


Requirements
------------
* Tomcat supported versions by this role:
  * 7.0
  * 8.0
  * 8.5
  * 9.0 (9.0.1 or later)
* CentOS/RHEL 7 or 8
* SELinux disabled


Example Playbook
----------------
```yaml
- hosts: webserver
  become: true
  vars:
    tomcat_version: 8.5.40

    tomcat_permissions_production: True

    tomcat_listen_address: 0.0.0.0
    tomcat_port_connector: 9095
    tomcat_port_shutdown: 9091
    tomcat_port_redirect: 9096
    tomcat_port_ajp: 9092
    tomcat_port_debug: 9090

    tomcat_user_roles:
      - tomcat
      - admin
      - manager
      - manager-gui
      - admin-gui
    tomcat_users:
      - username: "tomcat"
        password: "tomcatp@$$rd"
        roles: "tomcat,admin,manager,manager-gui,admin-gui"
    tomcat_sitename: "Testing"
    tomcat_aboutmessage: "Testiiiiiiiiiiiiiiiiiiiiiiiiiiiiiiiiiiiiiiiiiiiiiiiiiiiiiiiiing"
  roles:
    - role: tomcat
```

Role Variables
--------------
The main variable:
- `tomcat_version`: tomcat version to install

Some variables that require review:
- `tomcat_install_java`: True   
By default OpenJDK Java will be installed. Change it to "False" if you don't want OpenJDK Java to be installed by this role.
- `tomcat_java_version`: 1.8   
OpenJDK Java version to be installed. Default is "1.8". Currently, latest OpenJDK Java version is "11".
- `tomcat_install_path`: /opt   
Location in which tomcat will be installed. Default is "/opt".
- JVM memory management:   
You can set the minimum and maximum memory heap size with the following JVM -Xms and -Xmx variables as a percentage of the total system memory. For example, for a 2GB RAM system, using the default values: Xms=307m (15% of 2048MB), Xmx=1126m (55% of 2048MB).
  * `tomcat_jvm_memory_percentage_xms`: 15
  * `tomcat_jvm_memory_percentage_xmx`: 55   
- `tomcat_allow_manager_access_only_from_localhost`: False   
If set to "True", tomcat manager app will be accessible only from localhost for security reasons. (This behavior is default for Tomcat 8.5 and 9.0)
- `tomcat_allow_host_manager_access_only_from_localhost`: False   
If set to "True", tomcat host manager app will be accessible only from localhost for security reasons. (This behavior is default for Tomcat 8.5 and 9.0)
- `tomcat_users`: List of tomcat users to be created. See example for the expected format.
- `tomcat_debug_mode`: False   
Change it to "True" in order to configure tomcat to allow remote debugging. Default debug port is set to tcp/8000 (you can change it through the corresponding variable).

File permissions:
- `tomcat_permissions_production`: False  
For production installation, set this variable to "True" for more strict security. For development or low-security/more-ease installation, set this variable to "False". Default is "False".  
  * If set to "True", all tomcat files are owned by root with group tomcat. Owner has read/write privileges, group only has read and world has no permissions. The exceptions are the logs, temp and work directory that are owned by the tomcat user rather than root.  
  * If set to "False", all tomcat files are owned by tomcat with group tomcat. Owner and group has read/write privileges and world only has read permissions.
- `tomcat_webapps_auto_deployment`: True  
For better security, auto-deployment should be disabled and web applications should be deployed as exploded directories. If auto-deployment is disabled, set this to "False". This variable makes sense only for production installation (if tomcat_permissions_production is "True"). Default is "True".  
  * If set to "True", webapps subdirectory is owned by tomcat with group tomcat.
  * If set to "False", webapps subdirectory is owned by root with group tomcat.  
- `tomcat_permissions_ensure_on_every_run`: True  
If set to "True", file permissions are ensured on every playbook run. If set to "False", file permissions are set only when tomcat is installed (on first playbook run).
- `tomcat_prod_modes`:  
Lists Tomcat subdirectories, directory modes and file modes respectively. Permissions applied recursively.
```
tomcat_prod_modes:
 - ['bin', '2750', '0640']
 - ['conf', '2750', '0640']
 - ['lib', '2750', '0640']
 - ['logs', '0300', '0640']
 - ['temp', '0750', '0640']
 - ['work', '0750', '0640']
 - ['webapps', '0750', '0640']
```

Tomcat ports:
- `tomcat_port_connector`: 8080
- `tomcat_port_shutdown`: 8005
- `tomcat_port_redirect`: 8443
- `tomcat_port_ajp`: 8009
- `tomcat_port_debug`: 8000

Tomcat AJP:
- `tomcat_ajp_enabled`: False  
Set to "True" to enable AJP Connector.
- `tomcat_ajp_listen_address`: "::1"  
By default, the connector will listen on the loopback address. Unless the JVM is configured otherwise using system properties, the Java based connectors (NIO, NIO2) will listen on both IPv4 and IPv6 addresses when configured with either "0.0.0.0" or "::". The APR/native connector will only listen on IPv4 addresses if configured with "0.0.0.0" and will listen on IPv6 addresses (and optionally IPv4 addresses depending on the setting of ipv6v6only) if configured with "::".
- `tomcat_ajp_secret`: "my-@jp-s3cr3t"  
This attribute must be specified with a non-null, non-zero length value unless "secretRequired" is explicitly configured to be "false". Please change the default value to something secure.
- `tomcat_ajp_secret_required`: True  
Set to "False" to configure "secretRequired=False".

Some defaults (probably not requiring tampering):
- `tomcat_service_name`: tomcat
- `tomcat_service_enabled_on_startup`: True
- `tomcat_java_home`: /usr/lib/jvm/jre
- `tomcat_downloadURL`: https://<i></i>archive.apache.org/dist
- `tomcat_user`: tomcat
- `tomcat_group`: tomcat
- `tomcat_listen_address`: 0.0.0.0
- `tomcat_temp_download_path`: /tmp/ansibletomcattempdir
- `tomcat_systemd_config_path`: /etc/systemd/system

Custom templates for server.xml, users.xml, systemd service file, etc.:
- In case the default templates don't suit your needs, you can use your own custom templates by changing the following variables:
  * `tomcat_template_server`
  * `tomcat_template_users`
  * `tomcat_template_systemd_service`
  * `tomcat_template_setenv`
  * `tomcat_template_manager_context`
  * `tomcat_template_host_manager_context`

Optional variables (by default undefined):
- You can set custom user uid and group gid for homogeneity across multiple servers. For example:
  * `tomcat_user_uid`: 500
  * `tomcat_group_gid`: 500

In case of uninstallation:
- `tomcat_state`: absent
  * To uninstall tomcat that was installed using this role, set this variable to "absent". Default value is "present".
- `tomcat_uninstall_create_backup`: True   
By default, in a better safe than sorry basis, a backup tar archive will be created at "tomcat_install_path" before deletion.
- `tomcat_uninstall_remove_java`: False   
Change it to "True" to uninstall Java after tomcat is uninstalled.
- By default, tomcat user and group will be removed. Change to "False" to preserve them after tomcat is uninstalled.
  * `tomcat_uninstall_remove_user`: True
  * `tomcat_uninstall_remove_group`: True
- `tomcat_uninstall_remove_all`: False   
In order to override the above values and uninstall everything, set it to "True".

Variables for Disconnected remote environment:
- `tomcat_remote_is_disconnected`: False  
Change it to "True" if your remote host (managed host) is offline and cannot access internet.
