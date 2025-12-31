# Project Astraeus: Automated Two-Tier Infrastructure

Project Astraeus is an Infrastructure-as-Code (IaC) framework to deploy a distributed web architecture. It provisions an Nginx reverse proxy, a FastAPI application running in a VirtualEnv, and a PostgreSQL database managed via Docker.

---

## 1. Prerequisites & Installation

Before starting, ensure your host machine (macOS/Linux/Windows) has:

* Vagrant
* VirtualBox
* Ansible
* Python 3

Install Ansible on macOS via Homebrew:
```bash
brew install ansible
```

---

## 2. Initial Environment Setup

### Step 1 — Initialize Virtual Machines
From the project root:
```bash
vagrant up
```

### Step 2 — Secret Management (Ansible Vault)
Encrypt a string:
```bash
ansible-vault encrypt_string 'your_db_password' --name 'db_password'
```
Copy the resulting `!vault` block into `group_vars/all/secrets.yml`.

Optional: save the vault password to a file to avoid prompts:
```bash
echo "your_vault_password" > .vault_pass
```

### Step 3 — Configure SSH Access
The inventory uses relative paths. Ensure `inventories/staging/hosts` contains:
```ini
[all:vars]
ansible_user = vagrant
ansible_ssh_private_key_file = "{{ inventory_dir }}/../../.vagrant/machines/{{ vagrant_folder }}/virtualbox/private_key"
ansible_ssh_common_args = '-o StrictHostKeyChecking=no'
```

---

## 3. Deployment & Provisioning

### Run the full playbook
```bash
ansible-playbook -i inventories/staging/hosts site.yml --ask-vault-pass
# Or if using a password file:
ansible-playbook -i inventories/staging/hosts site.yml --vault-password-file .vault_pass
```

### Targeted deployments (by tag)
```bash
# Update only the FastAPI application
ansible-playbook -i inventories/staging/hosts site.yml --tags web --ask-vault-pass

# Update only Nginx configurations
ansible-playbook -i inventories/staging/hosts site.yml --tags nginx --ask-vault-pass
```

---

## 4. Core Architecture Concepts

* Modular roles: common, web, db, nginx — allows independent scaling and maintenance.
* Idempotency: Ansible enforces a desired state; repeated runs are safe.
* Service discovery: the Web tier discovers the DB IP via `hostvars`, e.g.
```jinja
host="{{ hostvars['dbserver']['ansible_host'] }}"
```

---

## 5. Command Reference

### Ad-hoc administration
| Action | Command |
| :--- | :--- |
| Check connectivity | `ansible all -i inventories/staging/hosts -m ping` |
| Restart all hosts | `ansible all -i inventories/staging/hosts -m reboot --become` |
| Gather system info | `ansible all -i inventories/staging/hosts -m setup` |
| Check disk space | `ansible all -i inventories/staging/hosts -a "df -h"` |

### Vault operations
| Action | Command |
| :--- | :--- |
| Encrypt a file | `ansible-vault encrypt secrets.yml` |
| View encrypted file | `ansible-vault view group_vars/all/secrets.yml` |
| Edit encrypted file | `ansible-vault edit group_vars/all/secrets.yml` |

---

## 6. Troubleshooting

* SSH Permission Denied: verify the path to the `.vagrant` private key and set permissions:
```bash
chmod 600 path/to/private_key
```
* Database authentication failed: if the password changed, remove the DB volume and restart the container:
```bash
vagrant ssh db -c "sudo rm -rf /opt/pgdata/* && sudo docker rm -f postgres_db"
```
* Nginx 502 Bad Gateway: check the FastAPI service and logs:
```bash
vagrant ssh web -c "sudo systemctl status astraeus"
vagrant ssh web -c "sudo journalctl -u astraeus -f"
```
## 7. Lifecycle Management (Power Operations)

Managing VM state is done with Vagrant. Use these commands to avoid data corruption.

### Shutting down
- Suspend (save RAM state)
```bash
vagrant suspend
```
- Halt (clean shutdown)
```bash
vagrant halt
```
- Destroy (remove VM and disks)
```bash
vagrant destroy
```

### Starting / resuming
- Start or recreate VMs
```bash
vagrant up
```
- Resume from suspend
```bash
vagrant resume
```
- Reload (re-provision the VM)
```bash
vagrant reload --provision
```

### Recommended workflow
- For temporary stop: use vagrant suspend
- For clean shutdown: use vagrant halt
- For configuration changes that require provisioning: vagrant reload --provision or vagrant up --provision

### Troubleshooting the astraeus systemd service

1) Verify the service file is present
```bash
vagrant ssh web -c "ls -l /etc/systemd/system/astraeus.service"
```
- If "No such file": the Ansible task did not place the template.

2) If the file exists, force Systemd to pick it up and start the service
```bash
vagrant ssh web -c "sudo systemctl daemon-reload"
vagrant ssh web -c "sudo systemctl start astraeus"
vagrant ssh web -c "sudo systemctl status astraeus --no-pager"
```

3) Confirm the Ansible systemd task includes daemon_reload
- Example task (roles/web/tasks/main.yml):
```yaml
- name: Start and enable Astraeus service
    systemd:
        name: astraeus
        state: restarted
        enabled: yes
        daemon_reload: yes
```

4) Check the virtualenv Python binary (common cause of failures)
```bash
vagrant ssh web -c "ls -l /opt/astraeus/venv/bin/python3"
```
- If missing, the virtualenv or pip tasks likely failed.

5) Force a re-run of the playbook to apply a corrected template
- Edit the template (for example add a comment), then:
```bash
ansible-playbook -i inventories/staging/hosts site.yml
# Or, if using a vault password file:
ansible-playbook -i inventories/staging/hosts site.yml --vault-password-file .vault_pass
```

6) If `systemctl status astraeus` shows "Loaded: not-found"
- Check the service filename and the path used by the Ansible template; correct naming or placement is required.

Notes:
- Run all commands from the project root unless otherwise noted.
- Use the vault options (--ask-vault-pass or --vault-password-file) consistent with the project configuration.
---

Notes:
* All commands assume execution from the project root and the `staging` inventory.
* Use `--ask-vault-pass` or `--vault-password-file .vault_pass` consistently based on your vault setup.