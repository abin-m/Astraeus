# Project Astraeus: Automated Two-Tier Infrastructure

Project Astraeus is an Infrastructure-as-Code (IaC) framework designed to deploy a distributed web architecture. It automates the provisioning of a high-performance Nginx reverse proxy, a FastAPI application running within an isolated Python environment, and a PostgreSQL database managed via Docker.

---

## Use Case: Reproducible Infrastructure

In a standard manual setup, "Configuration Drift" occurs when manual changes make servers inconsistent over time. Astraeus eliminates this by defining the entire environment as code.

**Key Use Cases:**
* **Rapid Scaling:** Deploy identical application stacks to multiple geographic regions by updating the inventory list.
* **Environment Parity:** Ensure that local development (Vagrant), staging, and production environments are identical, reducing "it works on my machine" bugs.
* **Disaster Recovery:** If a server becomes corrupted, the entire stack can be destroyed and reprovisioned in minutes using the playbook.



---

## Core Concepts

### 1. Modular Roles
The project uses **Ansible Roles** to separate concerns. Each role (`common`, `web`, `db`, `nginx`) is independent.
* **Concept:** Modularization allows teams to update the database engine (DB Role) without risking changes to the routing logic (Nginx Role).

### 2. Idempotency
Astraeus is built on the principle of idempotency. Running the playbook multiple times will not result in multiple installations. Ansible checks the current state against the desired state and only executes changes if necessary.
* **Example:** If Nginx is already installed and running, Ansible reports `ok` and moves to the next task without interrupting the service.

### 3. Service Discovery via Templating
The Web tier and Database tier are decoupled. The Web application does not have a hardcoded IP address. Instead, it "discovers" the database location during deployment.
* **Concept:** Using Jinja2 templates (`main.py.j2`), Ansible injects the database's private IP address into the application code at runtime.



---

## How to Update and Tweak

### Modifying Application Logic
To update the FastAPI code or add new endpoints:
1. Navigate to `roles/web/templates/main.py.j2`.
2. Add the Python code.
3. The `notify: restart astraeus` handler ensures the service reloads only when the file is updated.

### Changing System Dependencies
To add system-level packages (e.g., `curl`, `htop`, or `git`):
1. Open `roles/common/tasks/main.yml`.
2. Add the package name to the `apt` task list.
3. This ensures the dependency is present on every server in the cluster.

### Tuning Nginx Performance
To adjust the reverse proxy settings (e.g., adding a client request body limit):
1. Modify `roles/nginx/templates/astraeus.conf.j2`.
2. Run the playbook. Ansible will update the configuration and trigger a graceful Nginx restart.

---

## Project Structure Summary

| Component | Responsibility | Technical Implementation |
| :--- | :--- | :--- |
| **Common Role** | Security & Base Config | UFW Firewall, SSH hardening, system updates. |
| **Web Role** | Application Layer | Python venv, FastAPI, Systemd unit files. |
| **DB Role** | Data Layer | Docker Engine, PostgreSQL container, Data Volumes. |
| **Nginx Role** | Edge/Routing Layer | Reverse Proxying port 80 to 8000. |
| **Group Vars** | Secrets Management | Ansible Vault encrypted variables. |



---

## Deployment Commands

### Standard Deployment
To deploy the entire infrastructure:
```bash
ansible-playbook -i inventories/staging/hosts site.yml --ask-vault-pass