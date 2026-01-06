# Astraeus Infrastructure

A two-tier infrastructure deployment using Ansible, Vagrant, and VirtualBox. This project creates a web server running FastAPI behind Nginx and a separate database server with PostgreSQL.

## Quick Start

### Prerequisites
- Vagrant
- VirtualBox 
- Ansible
- Python 3

### Installation
Install Ansible:
```bash
# macOS
brew install ansible

# Ubuntu/Debian
sudo apt update && sudo apt install ansible

# RHEL/CentOS/Fedora
sudo dnf install ansible
```

Install required Ansible collections:
```bash
# Install individually
ansible-galaxy collection install community.docker
ansible-galaxy collection install ansible.posix

# Or install from requirements file
ansible-galaxy collection install -r requirements.yml
```

### Deployment
```bash
# Start VMs
vagrant up

# Deploy infrastructure
ansible-playbook -i inventories/staging/hosts site.yml
```

## Configuration

### Database Password
Set the database password in `group_vars/all/secrets.yml`:
```yaml
vault_db_password: "your_secure_password"
```

For production, encrypt this file:
```bash
ansible-vault encrypt group_vars/all/secrets.yml
```

## Architecture

**Web Server (192.168.56.10):**
- Nginx reverse proxy
- FastAPI application
- UFW firewall (SSH, HTTP, HTTPS)

**Database Server (192.168.56.11):**
- PostgreSQL in Docker
- UFW firewall (SSH, PostgreSQL from web server only)

## Common Commands

### Infrastructure Management
```bash
# Check status
vagrant status

# SSH into machines
vagrant ssh web
vagrant ssh db

# Run specific roles
ansible-playbook site.yml --tags web
ansible-playbook site.yml --tags nginx

# Test connectivity  
ansible all -i inventories/staging/hosts -m ping
```

### Service Management
```bash
# Restart services
vagrant ssh web -c "sudo systemctl restart astraeus"
vagrant ssh web -c "sudo systemctl restart nginx"

# Check logs
vagrant ssh web -c "sudo journalctl -u astraeus -f"
```

### VM Lifecycle
```bash
# Stop VMs (saves state)
vagrant suspend

# Clean shutdown
vagrant halt

# Start/resume VMs
vagrant up
vagrant resume

# Destroy VMs
vagrant destroy
```

## Troubleshooting

### Installation Issues
```bash
# If ansible-galaxy fails, try upgrading pip
pip3 install --upgrade pip

# Check if collections are installed
ansible-galaxy collection list | grep -E "(community.docker|ansible.posix)"

# Reinstall collections if needed
ansible-galaxy collection install --force community.docker ansible.posix
```

### SSH Issues
```bash
# Fix permissions
chmod 600 .vagrant/machines/*/virtualbox/private_key
```

### Database Problems
```bash
# Reset database
vagrant ssh db -c "sudo docker rm -f postgres_db"
vagrant ssh db -c "sudo rm -rf /opt/pgdata/*"
```

### Service Issues
```bash
# Check service status
vagrant ssh web -c "sudo systemctl status astraeus"

# Reload systemd and restart
vagrant ssh web -c "sudo systemctl daemon-reload"
vagrant ssh web -c "sudo systemctl restart astraeus"
```

### Application Testing
```bash
# Test web application
curl http://192.168.56.10/
curl http://192.168.56.10/health
curl http://192.168.56.10/db-test
```

## Security Features

- UFW firewall configured on both servers
- Database access restricted to web server only
- Security headers implemented in Nginx
- Server tokens hidden

## File Structure

```
Astraeus/
├── site.yml                 # Main playbook
├── ansible.cfg             # Ansible configuration
├── Vagrantfile             # VM definitions
├── inventories/
│   └── staging/hosts       # Server inventory
├── group_vars/
│   └── all/
│       ├── vars.yml        # Configuration variables
│       └── secrets.yml     # Sensitive data
└── roles/
    ├── common/             # Shared configurations
    ├── web/                # Web server setup
    ├── db/                 # Database setup
    └── nginx/              # Reverse proxy setup
```