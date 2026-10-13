# Cloud-1: Global Master Plan (Study, Implementation & Testing)

## Overview
Automated deployment of an Inception-like multi-container web infrastructure on a remote cloud server using **Ansible** and **Docker Compose**.

```text
Cloud-1/
├── Inception/
│   ├── docker-compose.yml          # Services stack (WordPress, MariaDB, PMA, Nginx)
│   └── nginx.conf                  # Reverse proxy, SSL, and routing
├── ansible/
│   ├── hosts.ini                   # Target server inventory (IP, SSH user, key)
│   ├── playbook.yml                # Main playbook orchestrating roles
│   └── roles/
│       ├── common/tasks/main.yml   # Base OS setup & UFW firewall
│       ├── docker/tasks/main.yml   # Docker Engine & Compose plugin
│       └── app/tasks/main.yml      # Deployment of Inception stack & launch
├── plan.md                         # This global master plan
├── README.md
└── LICENSE
```

---

## Phase 1: What to Study

### 1. Cloud & Remote Server Fundamentals
- **Instance Management**: AWS EC2 free tier (`t2.micro` / `t3.micro`), GCP (`e2-micro`), or Scaleway. Static/Elastic IPs, Security Groups, budgeting/shutting down instances.
- **SSH Access**: Key generation (`ed25519`), configuration (`~/.ssh/config`), and root access constraints required by 42 evaluation rules.
- **Host Firewall (UFW)**: Default deny policy, strictly allowing only ports:
  - `22` (SSH)
  - `80` (HTTP)
  - `443` (HTTPS)
  *(Port 3306 for MariaDB must never be exposed).*
- **Domain & DNS**: Dynamic DNS setup (DuckDNS, sslip.io, or nip.io) for TLS compatibility.

### 2. Ansible (Automation Tool)
- **Architecture**: Control node (your machine) vs Managed node (remote server over SSH, agentless via Python).
- **Inventories**: Structuring `hosts.ini` with `ansible_host`, `ansible_user`, and SSH private key path.
- **Roles & Tasks**: Organizing logic into `roles/<name>/tasks/main.yml`.
- **Core Modules**:
  - `ansible.builtin.apt`: Updating apt cache and installing packages.
  - `community.general.ufw`: Firewall configuration.
  - `ansible.builtin.copy` / `file`: Transferring directories and setting permissions.
  - `ansible.builtin.command` / `shell` or `community.docker.docker_compose_v2`: Controlling container lifecycle.
- **Idempotency**: Ensuring running the playbook multiple times results in `changed=0` if no state has drifted.

### 3. Docker & Docker Compose (Inception Core)
- **1 Container = 1 Process**:
  - `nginx`: Reverse proxy with TLS termination.
  - `wordpress`: WordPress application.
  - `mariadb`: Relational database backend.
  - `phpmyadmin`: Web database administration interface.
- **Networking**:
  - User-defined Docker bridge network.
  - Container-to-container name resolution (e.g., WordPress connects to `mariadb:3306`).
  - No host port mapping for MariaDB. Only Nginx binds to host `80:80` and `443:443`.
- **Data Persistence & Self-Healing**:
  - Named volumes for MariaDB (`/var/lib/mysql`) and WordPress (`/var/www/html`).
  - `restart: always` on all services to survive machine reboots.

### 4. Nginx & SSL/TLS
- **Reverse Proxy Routing**:
  - Root `/` forwarded to WordPress.
  - Path `/phpmyadmin` (or dedicated subdomain) forwarded to phpMyAdmin.
- **TLS Configuration**:
  - HTTP (port 80) redirects to HTTPS (port 443).
  - Valid SSL certificates (via Certbot / Let's Encrypt or automated self-signed certificates).

---

## Phase 2: Implementation Roadmap

```
Step 1: Local Inception Stack
   └── Write docker-compose.yml & nginx.conf in Inception/
Step 2: Ansible Role: common
   └── UFW firewall (22, 80, 443 only), system updates
Step 3: Ansible Role: docker
   └── Official Docker repository, engine, and compose plugin
Step 4: Ansible Role: app
   └── Transfer Inception/, generate SSL certs, start containers
Step 5: Cloud Provisioning & Orchestration
   └── Configure hosts.ini, bind domain, run playbook.yml
```

### Step 1: Inception Stack Implementation
- [ ] Create `Inception/docker-compose.yml` with the 4 isolated services (`mariadb`, `wordpress`, `phpmyadmin`, `nginx`).
- [ ] Configure named volumes for database and website files.
- [ ] Configure `Inception/nginx.conf`:
  - Port 80 redirect to 443.
  - Port 443 SSL termination with reverse proxy routing to WordPress and phpMyAdmin.

### Step 2: Role `common` (`ansible/roles/common/tasks/main.yml`)
- [ ] Update `apt` cache and upgrade packages.
- [ ] Install essential utilities (`curl`, `ufw`, `ca-certificates`).
- [ ] Configure UFW firewall:
  - Default incoming: `deny`
  - Default outgoing: `allow`
  - Allow rules: `22/tcp`, `80/tcp`, `443/tcp`
  - Enable UFW service.

### Step 3: Role `docker` (`ansible/roles/docker/tasks/main.yml`)
- [ ] Add Docker GPG key and official Ubuntu repository.
- [ ] Install `docker-ce`, `docker-ce-cli`, `containerd.io`, `docker-compose-plugin`.
- [ ] Enable and start Docker systemd service.
- [ ] Add target user to the `docker` group.

### Step 4: Role `app` (`ansible/roles/app/tasks/main.yml`)
- [ ] Create remote project directory (e.g. `/opt/inception`).
- [ ] Copy `Inception/` directory (`docker-compose.yml` and `nginx.conf`) to the remote host.
- [ ] Generate SSL certificates on the remote host (or configure Certbot).
- [ ] Run `docker compose up -d` to launch the stack.

### Step 5: Ansible Inventory & Playbook
- [ ] Configure `ansible/hosts.ini` with target IP, username, and SSH key.
- [ ] Configure `ansible/playbook.yml` to execute `common`, `docker`, and `app` roles in order.

---

## Phase 3: Testing & Verification Checklist

### 1. Functional Tests
- [ ] **WordPress Accessibility**: Access website via `https://<YOUR_DOMAIN_OR_IP>/` — complete WordPress initial setup or view existing site.
- [ ] **phpMyAdmin Integration**: Access `https://<YOUR_DOMAIN_OR_IP>/phpmyadmin` — log in with MariaDB credentials and verify database tables.
- [ ] **HTTP to HTTPS Redirection**: Test `http://<YOUR_DOMAIN_OR_IP>` and verify 301 redirect to HTTPS.

### 2. Security & Firewall Tests
- [ ] **Port Scan (from local machine)**:
  ```bash
  nmap -Pn <REMOTE_IP>
  ```
  *Expected Output*: Only ports `22`, `80`, and `443` are open.
- [ ] **Database Isolation**:
  ```bash
  nc -zv <REMOTE_IP> 3306
  ```
  *Expected Output*: Connection refused or timed out. MariaDB is strictly internal.

### 3. Resilience & Persistence Tests
- [ ] **Data Persistence**:
  1. Create a new test post/comment in WordPress.
  2. Create a test table or row in phpMyAdmin.
  3. Reboot the cloud instance: `sudo reboot`.
  4. Wait for instance reboot and reconnect.
  5. Check `docker ps` — all 4 containers must automatically be up (`restart: always`).
  6. Refresh the website — verify the test post and database data remain intact.

### 4. Portability & Idempotency Tests
- [ ] **Idempotency**:
  - Run the playbook a second time:
    ```bash
    ansible-playbook -i ansible/hosts.ini ansible/playbook.yml
    ```
  - Verify that no tasks fail and tasks report `ok` without breaking existing containers (`changed=0` where applicable).
- [ ] **Fresh Instance Portability**:
  - Spin up a clean, newly-created Ubuntu 22.04 LTS instance.
  - Run the playbook once against the new instance — verify that the entire infrastructure deploys with zero manual intervention.

---

## Phase 4: Defense & Evaluation Preparation
- [ ] **Root SSH Access**: Verify you can connect as `root` (or with your student login credentials) using SSH key.
- [ ] **Live Demonstration Readiness**:
  - Be ready to explain the purpose of each file and task.
  - Be ready to demonstrate container restart on reboot.
  - Be ready to demonstrate port blocking via `ufw status` and `nmap`.
- [ ] **Resource Cleanup**: Terminate/stop cloud instances after evaluation to avoid credit exhaustion.
