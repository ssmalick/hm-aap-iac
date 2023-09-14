# IAC

## Links

- [RedHat CoP Controller Configuration Collection](https://github.com/redhat-cop/controller_configuration/tree/devel)
- [Reference Architecture](https://access.redhat.com/documentation/en-us/reference_architectures/2021/html-single/deploying_ansible_automation_platform_2.1/index#config_as_code_using_webhooks)


### redhat-cop/controller_configuration Collection Roles

- [filetree_create](https://github.com/redhat-cop/controller_configuration/tree/devel/roles/filetree_create)
  Exports AAP Controller resources from a running instance on the filesystem.

- [filetree_read](https://github.com/redhat-cop/controller_configuration/tree/devel/roles/filetree_read)
  Reads AAP Controllers resources from filesystem into ansible variables ready to be used by dispatch role.

- [dispatch](https://github.com/redhat-cop/controller_configuration/tree/devel/roles/dispatch)
  Applies the resources on an AAP Controller instance


---

## Setup

```sh
### 1. Install collections required
cd .../iac
ansible-galaxy collection install -r collections/requirements.yml

### 2. Test
ansible-galaxy collection list | grep -i ansible.controller || echo NOTINSTALLED

### 3. Create ./vault_pass.txt file
echo "password" >> ./vault_pass.txt

### 4. Create vars/backup_credentials.yml
cat << EOF > vars/backup_credentials.yml
---
vault_controller_username: 'admin'
vault_controller_password: 'password'
vault_controller_hostname: '192.168.1.23'
vault_controller_validate_certs: false
EOF

### 5. Encrypt vars/backup_credentials.yml
ansible-vault --vault-password-file ./vault_pass.txt encrypt vars/backup_credentials.yml

### 6. Create restore_credentials.yml #TODO
cat << EOF > vars/backup_credentials.yml
---
vault_controller_username: 'admin'
vault_controller_password: 'password'
vault_controller_hostname: '192.168.1.23'
vault_controller_validate_certs: false
EOF

### 7. Encrypt backup_credentials.yml #TODO
ansible-vault --vault-password-file ./vault_pass.txt encrypt vars/restore_credentials.yml

### View vault encrypted file
# ansible-vault view --vault-pass-file vault_pass.txt vars/backup_credentials.yml
```

---
## Export / Backup AAP Resources

**Issues**
- credentials secret are empty
- Is inventory name unique across organization ?

```sh
cd .../iac

ansible-playbook playbooks/backup.yml \
  --vault-password-file ./vault_pass.txt \
  -e @./vars/backup_credentials.yml \
  -e output_path=/tmp/rhapp_filetree_output
 

# ### Export organizations and projects only
# ansible-playbook playbooks/backup.yml \
#   -e output_path=/tmp/rhapp_filetree_output \
#   -e '{input_tag: [organizations, projects]}' \
#   --vault-password-file ./vault_pass.txt

### Check ToDo
grep -R 'ToDo: ' '/tmp/rhapp_filetree_output'
```

**filetree_create role variables**

- `controller_api_plugin`: 
  Full path for the controller_api_plugin to be used.
  awx.awx.controller_api | ansible.controller.controller_api.
  Default: ansible.controller

- `organization_filter`:
  Exports only the objects belonging to the specified organization.

- `output_path`: 
  The path to the output directory where all the generated yaml files will be stored.
  Default: /tmp/filetree_output

- `input_tag`:
  The tags which are applied to the sub-roles.


---

## Import / Restore AAP resources

Note: **APP Resources generated in step #1 below need to be cleaned up before invoking `playbooks/restore.yml` in step #2.** See details below.


```sh
cd .../iac

### 1. Create filesystem expected by filetree_read role
ansible-playbook playbooks/restore_prep.yml \
  -e filletree_output_path=/tmp/rhapp_filetree_output \
  -e dir_orgs_vars=/tmp/rhapp_filetree_read_input \
  -e orgs=hm \
  -e env=nonprod

### 2. Restore
ansible-playbook playbooks/restore.yml \
  --vault-password-file ./vault_pass.txt \
  -e dir_orgs_vars=/tmp/rhapp_filetree_read_input \
  -e orgs=hm \
  -e env=nonprod \
  -e @./vars/restore_credentials.yml \
  -e controller_configuration_credentials_secure_logging=false
```


### AAP Resources cleaning up before restoring

**Notes**
- Project sync failure can cause restore playbook to fail but it seems to work on subsequent run.

#### **controller_settings.d/all_settings.yml**

The following may need to be commented out
- DEFAULT_EXECUTION_ENVIRONMENT: '2'  # THIS USED PK which is problematic. Can be handler after in UI
- SUBSCRIPTIONS_PASSWORD: ''
- SUBSCRIPTIONS_USERNAME: ''
- TOWER_URL_BASE: https://rhaap.lan
- REDHAT_PASSWORD: ''
- REDHAT_USERNAME: ''

#### **controller_users.d/all_users.yml**

- Comment out admin user since already set during install
- Update user passwords

#### **controller_instance_groups.d/all_instance_groups.yml**

The whole file can be commented out and handle the setup in the controller UI.

#### **controller_credentials.d/all_credentials.yml**

- password/token are not exported and will need to be provided
- credentials with same name and already available in the target instance can be commented out. For example:
  - Default Execution Environment Registry Credential
  - Demo Credential
- Handle credentials where `organization: ORGANIZATIONLESS` e.g. changed to `Default`

Resources provided by default and **all their reference** can be removed. Example
- Demo Inventory (controller_inventories.d/all_inventories.yml)
- Demo Project (controller_projects.d/all_projects.yml)
- Demo Job Template (controller_job_templates.d/all_job_templates.yml)
- Host provided by `Demo Inventory` (controller_hosts.d/all_hosts.yml)
- Default Execution Environment (controller_execution_environments.d/all_execution_environments.yml)
- Minimal Execution environment (controller_execution_environments.d/all_execution_environments.yml)
- Control Plane Execution environment (controller_execution_environments.d/all_execution_environments.yml)

#### **Handles all ToDo**

```sh
grep -R 'ToDo: ' /tmp/rhapp_filetree_read_input/

all_workflow_job_templates.yml:   organization: 'ToDo: The WF ''TF httpd demo workflow'' must belong to an organization'
all_workflow_job_templates.yml:   organization: 'ToDo: The WF ''TF httpd demo workflow'' must belong to an organization'
all_workflow_job_templates.yml:   organization: 'ToDo: The WF ''TF httpd demo workflow'' must belong to an organization'
all_workflow_job_templates.yml:   organization: 'ToDo: The WF ''TF httpd demo workflow'' must belong to an organization'
```