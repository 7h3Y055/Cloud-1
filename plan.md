# Cloud-1: Minimalist Master Plan (Maximum Compression)

## Philosophy
The 42 subject allows **official Docker images** and requires **Ansible with roles** and a **docker-compose.yml**.  
To keep the repository as small, clean, and maintainable as possible, we eliminate all unnecessary boilerplate (no custom Dockerfiles, no redundant script folders, no deep variable hierarchies).

---

## 1. Minimal File Tree (Total: 7 Essential Files)

```text
Cloud-1/
├── docker-compose.yml          # [1] All 4 services using official images
├── nginx.conf                  # [2] Reverse proxy routing + SSL config
├── playbook.yml                # [3] Main Ansible playbook
├── hosts.ini                   # [4] Server inventory (IPs, SSH user)
└── roles/
    ├── common/
    │   └── tasks/main.yml      # [5] UFW firewall (22, 80, 443 only) & packages
    ├── docker/
    │   └── tasks/main.yml      # [6] Install Docker Engine & Compose plugin
    └── app/
        └── tasks/main.yml      # [7] Copy files & start docker compose
```

### What We Cut Out
- ❌ **`docker/*` subdirectories**: Official images (`nginx:alpine`, `wordpress:fpm-alpine`, `mariadb:10`, `phpmyadmin:latest`) are used directly in `docker-compose.yml`. No Dockerfiles needed.
- ❌ **`scripts/` folder**: Replaced by standard one-line execution (`ansible-playbook -i hosts.ini playbook.yml`).
- ❌ **`group_vars/` folder**: Variables placed directly inside `hosts.ini` or `playbook.yml`.
- ❌ **Excess role folders** (`handlers/`, `defaults/`, `vars/`, `meta/`): Everything is consolidated into `tasks/main.yml` within each role.

---

## 2. What You Need to Study (Concise List)

1. **UFW (Firewall)**:
   - Command/module rules: default deny, allow `22/tcp`, `80/tcp`, `443/tcp`.
2. **Ansible Minimal Essentials**:
   - `hosts.ini` format (defining `ansible_host`, `ansible_user`, `ansible_ssh_private_key_file`).
   - `apt` module (update cache, install packages).
   - `copy` / `template` module (transfer `docker-compose.yml` and `nginx.conf`).
   - `command` module (run `docker compose up -d`).
   - Understanding roles (`roles/<name>/tasks/main.yml`).
3. **Docker Compose**:
   - Bridge network: Container-to-container communication (WordPress & phpMyAdmin -> MariaDB on port 3306).
   - Port forwarding: Expose **only** ports 80 and 443 on the `nginx` container. MariaDB has **no** host port mapping.
   - Volumes: Named volumes for MariaDB (`/var/lib/mysql`) and WordPress (`/var/www/html`).
   - Restart policy: `restart: always` on all services for reboot survival.
4. **Nginx Reverse Proxy**:
   - Routing: Map root (`/`) to WordPress and `/phpmyadmin` (or subdomain) to phpMyAdmin.
   - TLS: Self-signed certificate or Let's Encrypt (DuckDNS).
   - Redirection: HTTP (80) -> HTTPS (443).

---

## 3. Implementation Steps (A to Z)

### Step 1: Clean Up & Prepare the Repository
Remove all unnecessary boilerplate folders (`docker/mariadb`, `docker/wordpress`, `scripts/`, etc.). Keep only the root files and the 3 minimal roles.

### Step 2: Write `docker-compose.yml`
Define the 4 containers:
- `mariadb`: Official image, environment variables, named volume, internal network.
- `wordpress`: Official image, connects to `mariadb`, named volume, internal network.
- `phpmyadmin`: Official image, connects to `mariadb`, internal network.
- `nginx`: Official image, mounts `nginx.conf` and SSL certificates, exposes `80:80` and `443:443`.

### Step 3: Write `nginx.conf`
- Listen on 80: redirect to 443.
- Listen on 443 SSL: route traffic to WordPress (`wordpress:9000` or `wordpress:80`) and phpMyAdmin (`phpmyadmin:80`).

### Step 4: Write the 3 Ansible Roles
- **`roles/common/tasks/main.yml`**:
  - Update apt cache.
  - Set UFW to deny incoming, allow 22, 80, 443. Enable UFW.
- **`roles/docker/tasks/main.yml`**:
  - Install prerequisites (`ca-certificates`, `curl`).
  - Add Docker apt repository and install `docker-ce`, `docker-compose-plugin`.
  - Start and enable Docker service.
- **`roles/app/tasks/main.yml`**:
  - Create app directory (e.g., `/opt/cloud1`).
  - Copy `docker-compose.yml` and `nginx.conf`.
  - Generate self-signed SSL certs (or obtain Let's Encrypt).
  - Run `docker compose up -d`.

### Step 5: Write `hosts.ini` and `playbook.yml`
- `hosts.ini`: Define server IP and SSH settings.
- `playbook.yml`: Apply `common`, `docker`, and `app` roles in order.

### Step 6: Deploy & Verify
```bash
ansible-playbook -i hosts.ini playbook.yml
```
Verify:
1. `nmap <IP>`: Only ports 22, 80, 443 open.
2. Web browser: Visit `https://<IP>`, test WordPress and phpMyAdmin.
3. Persistence test: Run `sudo reboot` on the server. Verify site recovers with all data intact.
