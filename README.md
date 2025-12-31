# Project Astraeus: Automated Two-Tier Infrastructure

Project Astraeus is an Infrastructure-as-Code (IaC) framework that provisions a distributed web architecture. It automates the deployment of an Nginx reverse proxy, a FastAPI application running in a Python virtual environment, and a PostgreSQL database managed with Docker.

---

## 1. Prerequisites & Installation

Before starting, ensure your host machine (macOS / Linux / Windows) has the following installed:

- Vagrant (VM lifecycle)
- VirtualBox (hypervisor)
- Ansible (automation)
    ```bash
    # macOS (Homebrew)
    brew install ansible
    ```
- Python 3 (for local Ansible execution)



## 2. Initial Environment Setup

### Step 1 — Initialize Virtual Machines
From the project root, start the Vagrant nodes (`web` and `db`):
```bash
vagrant up
```

### Step 2 — Secret Management (Ansible Vault)
Create an encrypted DB password and add it to your secrets file:
```bash
ansible-vault encrypt_string 'your_db_password' --name 'db_password'
```
Paste the resulting encrypted string into `inventories/staging/group_vars/all/secrets.yml`.

### Step 3 — Configure SSH Access
Ensure your `inventories/staging/hosts` uses relative paths to Vagrant private keys. Example variables:
```ini
[all:vars]
ansible_user=vagrant
ansible_ssh_private_key_file={{ inventory_dir }}/../../.vagrant/machines/{{ vagrant_folder }}/virtualbox/private_key
ansible_ssh_common_args='-o StrictHostKeyChecking=no'
```


## 3. Deployment & Provisioning

Run the full playbook (Common, DB/Docker, FastAPI, Nginx):
```bash
ansible-playbook -i inventories/staging/hosts site.yml --ask-vault-pass
```

Targeted deploy (only web role):
```bash
ansible-playbook -i inventories/staging/hosts site.yml --tags web --ask-vault-pass
```


## 4. Core Architecture Concepts

- Modular Roles: `common`, `web`, `db`, and `nginx` separate concerns.
- Idempotency: Ansible enforces desired state; repeated runs are safe.
- Service Discovery: Web tier discovers DB IP via Jinja2, e.g.:
    `host="{{ hostvars['dbserver']['ansible_host'] }}"`


## 5. Management Command Reference

| Action | Command |
| :--- | :--- |
| Check Connectivity | `ansible all -i inventories/staging/hosts -m ping` |
| Reboot All Hosts | `ansible all -i inventories/staging/hosts -m reboot --become` |
| Check Uptime | `ansible all -i inventories/staging/hosts -a "uptime"` |
| View Secrets | `ansible-vault view inventories/staging/group_vars/all/secrets.yml` |
| Edit Secrets | `ansible-vault edit inventories/staging/group_vars/all/secrets.yml` |



## 6. Troubleshooting

- SSH Permission Denied: Verify the `.vagrant` private key path and set permissions to `600`.
    ```bash
    chmod 600 .vagrant/machines/<vm>/virtualbox/private_key
    ```
- Database Authentication Failed: If the container password and Ansible secret diverge, remove the DB volume and recreate:
    ```bash
    vagrant ssh db -c "sudo rm -rf /opt/pgdata/* && sudo docker rm -f postgres_db"
    ```
- Nginx 502 Bad Gateway: Indicates the FastAPI app is not responding on port 8000. Check service status:
    ```bash
    vagrant ssh web -c "sudo systemctl status astraeus"
    ```

---