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
├── site.yml
│
├── group_vars/
│   ├── all.yml
│   └── web/
│       ├── vars.yml
│       └── vault.yml
│
├── roles/
│   ├── common/
│   ├── docker/
│   └── nginx/
│
└── templates/
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
ansible-galaxy collection install community.docker
```

---

### 3. Configure Inventory

```ini
[web]
web-server ansible_host=<PUBLIC_IP>

[all:vars]
ansible_user=ec2-user
ansible_ssh_private_key_file=~/your-key.pem
```

---

### 4. Configure Ansible

`ansible.cfg`

```ini
[defaults]
inventory = inventory.ini
host_key_checking = False
vault_password_file = .vault_pass
```

---

## 🔧 Common Role

### `roles/common/tasks/main.yml`

```yaml
- name: Update package cache
  yum:
    update_cache: true

- name: Install common packages
  yum:
    name: "{{ common_packages }}"
    state: present

- name: Set timezone
  timezone:
    name: "{{ timezone }}"

- name: Create deploy user
  user:
    name: deploy
    groups: wheel
    shell: /bin/bash
```

---

## 🐳 Docker Role

### Defaults

```yaml
docker_app_image: nginx
docker_app_tag: latest
docker_app_name: myapp
docker_app_port: 8080
docker_container_port: 80
```

### Tasks

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
```

---

## 🌐 Nginx Role

### Defaults

```yaml
nginx_http_port: 80
nginx_upstream_port: 8080
```

### Template: `app-proxy.conf.j2`

```nginx
upstream docker_app {
    server 127.0.0.1:{{ nginx_upstream_port }};
}

server {
    listen {{ nginx_http_port }};

    location / {
        proxy_pass http://docker_app;
    }
}
```

### Tasks

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
```

---

## 🔐 Ansible Vault

### Create Vault

```bash
ansible-vault create group_vars/web/vault.yml
```

### Example

```bash
vault_docker_username: your-username
vault_docker_password: your-token
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
