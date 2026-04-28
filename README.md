# 🚀 Ansible Docker + Nginx Automation Project

## 📌 Overview

This project demonstrates a **production-style DevOps automation workflow** using Ansible.

It automates:

* Docker installation
* Container deployment
* Nginx reverse proxy setup
* Secure credential handling using Ansible Vault

👉 With a single command, a fresh server becomes a **fully running application stack**.

---

## 🧱 Tech Stack

* Ansible
* Docker
* Nginx
* AWS EC2
* Linux (Amazon Linux / Ubuntu)
* Ansible Vault

---

## 🏗️ Architecture

```
Control Node (Ansible)
        |
        | SSH
        ↓
Managed Node (EC2)
   ├── Docker Container (App running on port 8080)
   └── Nginx Reverse Proxy (Port 80 → 8080)
```

---

## 📂 Project Structure

```
ansible-docker-project/
│
├── ansible.cfg
├── inventory.ini
├── site.yml                            # Master playbook
│
├── group_vars/
│   ├── all.yml                         # Common variables
│   └── web/
│       └── vault.yml                   # Encrypted Docker Hub credentials
│
└── roles/
    ├── common                          # Shared setup for all servers
    │    └── tasks/main.yml
    ├── docker                          # Docker installation and container management
    │      ├── tasks/main.yml
    │      ├── handlers/main.yml
    │      └── defaults/main.yml
    └── nginx                           # Nginx reverse proxy
          ├── tasks/main.yml
          ├── templates
          │     └──app-proxy.conf.j2
          ├── handlers/main.yml
          └── defaults/main.yml

```

---

## ⚙️ Setup Instructions

### 1. Clone Repo

```bash
git clone https://github.com/Shibnath27/ansible-docker-project.git
cd ansible-docker-project

```

---

### 2. Install Required Collection

```bash
ansible-galaxy init roles/common
ansible-galaxy init roles/docker
ansible-galaxy init roles/nginx
```

---

### 3. Configure Inventory

`inventory.ini`

```ini
[web]
web-server ansible_host=<PUBLIC_IP>

[all:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=~/.ssh/ansible-practice
ansible_python_interpreter=/usr/bin/python3
```

---

### 4. Configure Ansible

`ansible.cfg`

```ini
[defaults]
inventory = inventory.ini
host_key_checking = False
vault_password_file = .vault_pass
remote_user = ec2-user
private_key_file = ~/.ssh/ansible-practice

```

---

## 🔧 Common Role

The common role runs on every server -- baseline packages and setup.
### `roles/common/tasks/main.yml`

```yaml
---
- name: Update package cache
  yum:
    update_cache: true
  tags: common

- name: Install common packages
  yum:
    name: "{{ common_packages }}"
    state: present
  tags: common

- name: Set hostname
  hostname:
    name: "{{ inventory_hostname }}"
  tags: common

- name: Set timezone
  timezone:
    name: "{{ timezone }}"
  tags: common

- name: Create deploy user
  user:
    name: deploy
    groups: sudo
    shell: /bin/bash
    state: present
  tags: common
```
(Use apt instead of yum if your instances run Ubuntu)
### `group_vars/all.yml:`

```yaml
---
timezone: Asia/Kolkata
project_name: devops-app
app_env: development
common_packages:
  - vim
  - curl
  - wget
  - git
  - htop
  - tree
  - jq
  - unzip

```
---

## 🐳 Docker Role

This role installs Docker, starts the service, pulls images, and runs containers.
### `roles/docker/defaults/main.yml`:

```yaml
docker_app_image: nginx
docker_app_tag: latest
docker_app_name: myapp
docker_app_port: 8080
docker_container_port: 80
```

### `roles/docker/tasks/main.yml`

```yaml
- name: Install Docker
  yum:
    name: docker
    state: present

- name: Start Docker
  service:
    name: docker
    state: started
    enabled: true

- name: Login to Docker Hub
  community.docker.docker_login:
    username: "{{ vault_docker_username }}"
    password: "{{ vault_docker_password }}"
  when: vault_docker_username is defined

- name: Pull image
  community.docker.docker_image:
    name: "{{ docker_app_image }}"
    tag: "{{ docker_app_tag }}"
    source: pull

- name: Run container
  community.docker.docker_container:
    name: "{{ docker_app_name }}"
    image: "{{ docker_app_image }}:{{ docker_app_tag }}"
    state: started
    restart_policy: always
    ports:
      - "{{ docker_app_port }}:{{ docker_container_port }}"

- name: Wait for container to be healthy
  uri:
    url: "http://localhost:{{ docker_app_port }}"
    status_code: 200
  retries: 5
  delay: 3
  register: health_check
  until: health_check.status == 200
```
Tag all tasks with docker.

### `roles/docker/handlers/main.yml`:

```yaml
---
- name: Restart Docker
  service:
    name: docker
    state: restarted
```
Install the required Ansible collection (needed for community.docker modules):
```bash
ansible-galaxy collection install community.docker
```
---

## 🌐 Nginx Role

This role installs Nginx and configures it as a reverse proxy to the Docker container.
### `roles/nginx/defaults/main.yml`

```yaml
---
nginx_http_port: 80
nginx_upstream_port: 8080
nginx_server_name: "_"
```

### Template: `roles/nginx/templates/app-proxy.conf.j2`

```nginx
# Reverse Proxy to Docker Container -- Managed by Ansible
upstream docker_app {
    server 127.0.0.1:{{ nginx_upstream_port }};
}

server {
    listen {{ nginx_http_port }};
    server_name {{ nginx_server_name }};

    location / {
        proxy_pass http://docker_app;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /health {
        access_log off;
        return 200 'OK';
        add_header Content-Type text/plain;
    }

{% if app_env == 'production' %}
    access_log /var/log/nginx/{{ project_name }}_access.log;
    error_log /var/log/nginx/{{ project_name }}_error.log;
{% else %}
    access_log /var/log/nginx/{{ project_name }}_access.log;
    error_log /var/log/nginx/{{ project_name }}_error.log debug;
{% endif %}
}
```

### Tasks `roles/nginx/tasks/main.yml`

```yaml
- name: Install Nginx
  yum:
    name: nginx
    state: present

- name: Deploy config
  template:
    src: app-proxy.conf.j2
    dest: /etc/nginx/conf.d/app.conf
  notify: Reload Nginx

- name: Start Nginx
  service:
    name: nginx
    state: started
    enabled: true

- name: Test Nginx configuration
  command: nginx -t
  changed_when: false
```
Tag all tasks with nginx.
### `roles/nginx/handlers/main.yml`

```yaml
---
- name: Reload Nginx
  service:
    name: nginx
    state: reloaded

- name: Restart Nginx
  service:
    name: nginx
    state: restarted
```
---

## 🔐 Ansible Vault

### 1. Create Vault

```bash
ansible-vault create group_vars/web/vault.yml
```

### Example

```bash
vault_docker_username: your-username
vault_docker_password: your-token
```
### 2. Create a vault password file for convenience:

```bash
echo "YourVaultPassword" > .vault_pass
chmod 600 .vault_pass
echo ".vault_pass" >> .gitignore
```
### 3. Reference it in ansible.cfg:
```ini
[defaults]
inventory = inventory.ini
host_key_checking = False
vault_password_file = .vault_pass
```
---

## Write the Master Playbook

### `site.yml`

```bash
---
- name: Apply common configuration
  hosts: all
  become: true
  roles:
    - common
  tags: common

- name: Install Docker and run containers
  hosts: web
  become: true
  roles:
    - docker
  tags: docker

- name: Configure Nginx reverse proxy
  hosts: web
  become: true
  roles:
    - nginx
  tags: nginx
```
---

## 🚀 Deployment

### Dry Run

```bash
ansible-playbook site.yml --check --diff
```

### Full Deployment

```bash
ansible-playbook site.yml
```
### Use tags for selective execution:

```bash
# Only set up Docker and containers
ansible-playbook site.yml --tags docker

# Only update Nginx config
ansible-playbook site.yml --tags nginx

# Skip common setup
ansible-playbook site.yml --skip-tags common
```
---

## 🎯 Verification

### Check Container

```bash
docker ps
```

### Test App Directly

```bash
curl http://<server-ip>:8080
```

### Test via Nginx

```bash
curl http://<server-ip>
```

---
## 🎯 So I actually built THIS architecture:

```scss
             ┌──────────────────────────┐
             │        USER              │
             └──────────┬───────────────┘
                        │
        ┌───────────────┴───────────────┐
        │                               │
   Port 80                         Port 8080
        │                               │
        ▼                               ▼
 Host Nginx                    Docker container
 (Reverse Proxy)               (Nginx inside)
        │
        ▼
 localhost:8080
        │
        ▼
 Docker container
 
```
## 🔄 Re-Deploy with Different App

```bash
ansible-playbook site.yml --tags docker \
-e "docker_app_image=httpd docker_app_name=apache-app"
```

---

## 🧠 Key Concepts Used

| Day | Concept                      |
| --- | ---------------------------- |
| 01  | Inventory, SSH, Ad-hoc       |
| 02  | Playbooks, Modules, Handlers |
| 03  | Variables, Facts, Loops      |
| 04  | Roles, Templates, Vault      |
| 05  | Full Project Integration     |

---

## 💡 Key Learnings

* Infrastructure can be fully automated with Ansible
* Roles provide scalable architecture
* Templates enable dynamic configs
* Vault ensures secure secret management
* Idempotency guarantees consistent deployments

---

## 🚀 Future Improvements

* SSL (Let's Encrypt + Certbot)
* Docker Compose multi-container setup
* Monitoring (Prometheus + Grafana)
* Logging (ELK Stack)
* CI/CD Integration

---

## 📌 Conclusion

This project demonstrates how to move from:
➡️ Basic Ansible usage
➡️ To full production-style automation

---

## 🔗 Author

Shibnath

DevOps | Cloud | Automation

---
