# Deploying Passly on Oracle Cloud Infrastructure (OCI) Always Free Tier

This guide provides an end-to-end blueprint for deploying and maintaining the **Passly** backend API on Oracle Cloud Infrastructure (OCI) Always Free resources. It is specifically designed to give an AI coding agent or systems engineer all necessary technical details, operational gotchas, and architectural choices to provision a production-grade instance from scratch.

---

## 1. OCI Always Free Resource Overview

Oracle Cloud provides a generous Always Free tier that is well-suited for running Passly:

- **Compute Options**:
  - **AMD Micro**: Up to 2× `VM.Standard.E2.1.Micro` instances (each with 1/8 OCPU, 1 GB RAM, AMD EPYC).
  - **Ampere A1 (ARM)**: Up to 4 OCPUs and 24 GB RAM, allocatable across 1 to 4 instances (e.g. 1× 2 OCPU / 12 GB RAM instance).
- **Storage**: Up to 200 GB total block volume storage across all compute instances.
- **Networking**: 1 Virtual Cloud Network (VCN), public/private subnets, 1 reserved public IPv4 address, and generous outbound bandwidth (10 TB/month).

> [!NOTE]
> While Ampere A1 offers significantly more memory and compute, the AMD Micro shape (`VM.Standard.E2.1.Micro`) is universally available in all regions. This guide details sizing and memory constraints specifically tuned to fit within the 1 GB footprint of a Micro VM, making it universally applicable.

---

## 2. Architecture Models

Choose between the **Standard Single-Host Architecture** (recommended for simplicity) or the **Two-Tier DMZ Architecture** (for advanced defense-in-depth).

```
[ Model A: Single-Host (Default) ]
Internet ---> [ Public Edge / Passly VM ] (Reverse Proxy + TLS + Passly API + MySQL)

[ Model B: Two-Tier DMZ (Optional, High-Security) ]
Internet ---> [ Edge Reverse Proxy VM (Public IP) ] 
                     | (VCN Private Subnet: 10.0.0.0/24)
                     v
              [ Passly Backend VM (Private-Only, No Public IP) ]
```

---

## 3. Baseline Deployment: Single-Host Setup

In this model, a single VM handles TLS termination, the Passly CakePHP application, and the database.

### Step 1: OCI Network & Compute Provisioning

1. **Virtual Cloud Network (VCN)**:
   - CIDR block: `10.0.0.0/16` (or `10.0.0.0/24`).
   - Create an **Internet Gateway** and attach it to the VCN route table for `0.0.0.0/0`.
2. **Security List Rules (Ingress)**:
   - TCP port `22` (SSH) from your administrative IP range.
   - TCP port `80` (HTTP for ACME challenge / HTTPS redirects) from `0.0.0.0/0`.
   - TCP port `443` (HTTPS) from `0.0.0.0/0`.
3. **Launch Compute Instance**:
   - Shape: `VM.Standard.E2.1.Micro` (or `VM.Standard.A1.Flex`).
   - Assign a **Reserved Public IPv4** address.
   - Add your administrative SSH public key.
4. **DNS Setup**:
   - Point an `A` record (e.g. `passly.example.com`) to the reserved public IP before configuring TLS certificates.

---

### Step 2: System Dependencies & Database Tuning

On a 1 GB RAM Micro instance, MySQL memory allocation must be strictly bounded to prevent the Out-Of-Memory (OOM) killer from terminating processes.

#### 1. Configure Swap (Mandatory for 1 GB RAM)
```bash
sudo fallocate -l 2G /var/swapfile
sudo chmod 600 /var/swapfile
sudo mkswap /var/swapfile
sudo swapon /var/swapfile
echo '/var/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

#### 2. Install Software Stack
- **Web Server**: Nginx
- **PHP**: PHP 8.2 or 8.5 with extensions: `php-fpm`, `php-pdo_mysql`, `php-intl`, `php-gd`, `php-mbstring`, `php-xml`, `php-curl`, `php-gmp`
- **GnuPG**: `gnupg` (required for server-side OpenPGP operations)
- **Database**: MySQL 8.0 or MariaDB

#### 3. Low-Memory MySQL 8.0 Configuration
Add to `/etc/mysql/conf.d/passly-lowmem.cnf` (or `my.cnf`):
```ini
[mysqld]
bind-address = 127.0.0.1
performance_schema = OFF
innodb_buffer_pool_size = 64M
max_connections = 16
tmp_table_size = 16M
max_heap_table_size = 16M
```
Restart MySQL and create the database:
```sql
CREATE DATABASE passly CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'passly'@'127.0.0.1' IDENTIFIED BY '<STRONG_DATABASE_PASSWORD>';
GRANT ALL PRIVILEGES ON passly.* TO 'passly'@'127.0.0.1';
FLUSH PRIVILEGES;
```

---

### Step 3: Passly Application Setup

1. **Deploy Source**:
   Clone `Svaroh/passly_api` into `/var/www/passly`:
   ```bash
   git clone --branch main https://github.com/Svaroh/passly_api.git /var/www/passly
   cd /var/www/passly
   composer install --no-dev --optimize-autoloader
   ```

2. **Generate Server OpenPGP Keys**:
   The server needs its own GPG keypair for cryptographic handshakes:
   ```bash
   export GNUPGHOME="/var/lib/passly/.gnupg"
   sudo mkdir -p "$GNUPGHOME"
   sudo gpg --batch --gen-key <<EOF
   Key-Type: RSA
   Key-Length: 2048
   Subkey-Type: RSA
   Subkey-Length: 2048
   Name-Real: Passly Server Key
   Name-Email: passly@example.com
   Expire-Date: 0
   %no-protection
   %commit
   EOF
   sudo chown -R www-data:www-data /var/lib/passly
   sudo chmod -R 700 "$GNUPGHOME"
   ```

3. **Runtime Configuration (`passbolt.php`)**:
   Configure `/var/lib/passly/config/passbolt.php` (or copy from `config/passbolt.default.php`):
   ```php
   return [
       'App' => [
           'fullBaseUrl' => 'https://passly.example.com',
       ],
       'passbolt' => [
           'plugins' => [
               'openpgp' => [
                   'serverKey' => [
                       'fingerprint' => '<SERVER_KEY_FINGERPRINT>',
                       'publicKey' => '/var/lib/passly/config/gpg/serverkey.asc',
                       'privateKey' => '/var/lib/passly/config/gpg/serverkey_private.asc',
                   ],
               ],
           ],
           'security' => [
               'smtpSettings' => [
                   'endpointsDisabled' => true,
               ],
           ],
       ],
       'Session' => [
           'cookie' => 'passly_session',
           'defaults' => 'database',
           'handler' => [
               'engine' => 'DatabaseSession',
               'model' => 'Sessions',
           ],
       ],
       'Datasources' => [
           'default' => [
               'host' => '127.0.0.1',
               'username' => 'passly',
               'password' => '<STRONG_DATABASE_PASSWORD>',
               'database' => 'passly',
               'encoding' => 'utf8mb4',
           ],
       ],
   ];
   ```

> [!IMPORTANT]
> **Database-backed Sessions**: Always use `DatabaseSession` rather than file-backed PHP sessions. On small VM instances, this ensures user sessions survive PHP-FPM restarts or worker recycling without dropping active users.

4. **Initialize Migrations & Create First Admin**:
   ```bash
   sudo -u www-data bin/cake migrations migrate
   sudo -u www-data bin/cake passbolt register_user -u admin@example.com -f Admin -l User -r admin
   ```
   Save the emitted registration link.

---

### Step 4: Nginx Reverse Proxy & TLS

Configure `/etc/nginx/sites-available/passly.conf`:

```nginx
server {
    listen 80;
    server_name passly.example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name passly.example.com;

    ssl_certificate /etc/letsencrypt/live/passly.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/passly.example.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    root /var/www/passly/webroot;
    index index.php;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_pass unix:/run/php/php8.2-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param HTTPS on;
    }

    location ~* ^/(favicon\.ico|robots\.txt)$ {
        expires 30d;
        access_log off;
    }
}
```

Verify and restart:
```bash
sudo nginx -t && sudo systemctl reload nginx
```

---

## 4. Optional Security Architecture: Two-Tier DMZ Setup

For environments with stricter security requirements, Passly can be partitioned into two distinct hosts:

1. **Edge VM (`edge-proxy`)**:
   - Has the public reserved IPv4 address.
   - Terminates TLS via Let's Encrypt / ACME.
   - Acts as a reverse proxy forwarding requests to the private backend.
   - Acts as an SSH Jump Host (Bastion) for managing the backend.
   - Runs raw TCP relays for outbound services needed by the backend.
2. **Passly VM (`passly-backend`)**:
   - Resides in the private VCN subnet (`10.0.0.0/24`) with **no public IP**.
   - Accessible only via private IP (`10.0.0.20`) from the Edge VM.
   - Holds the database, password records, and server private GPG keys.

### Benefits of the Two-Tier Architecture

- **Reduced Attack Surface**: The database and application host are not directly reachable from the public internet. Port scans and zero-day exploits targeting web services hit only the reverse proxy.
- **Protected Secrets**: In the event of a reverse-proxy exploit, the attacker gains no direct access to the database or private encryption keys.
- **Centralized Edge Defense**: The edge VM can run rate-limiting, WAF rules, or DDoS mitigation without affecting backend CPU.

### Managing a Private-Only VM Behind the Edge Proxy

#### 1. SSH Jump Host Access (`ProxyCommand`)
Connect from your local workstation through the edge VM to the private VM:

```bash
ssh -o ProxyCommand='ssh -W %h:%p admin@<EDGE_PUBLIC_IP>' \
    admin@<PASSLY_PRIVATE_IP>
```

To configure in your local `~/.ssh/config`:
```text
Host passly-edge
    HostName <EDGE_PUBLIC_IP>
    User admin
    IdentityFile ~/.ssh/id_ed25519

Host passly-backend
    HostName <PASSLY_PRIVATE_IP>
    User admin
    ProxyJump passly-edge
    IdentityFile ~/.ssh/id_ed25519
```

#### 2. Outbound Network Connectivity for the Private Host

Because the private VM has no public IP or default gateway to the internet, it cannot perform outbound operations (such as sending emails via external SMTP or pulling Git updates) unless routing is provided.

There are two ways to solve this:
- **OCI NAT Gateway (Cloud-native approach)**: Add an OCI NAT Gateway to your VCN and route `0.0.0.0/0` from the private subnet through it.
- **Socket-Activated TCP Relays on Edge VM (Minimalist approach)**:
  Run lightweight TCP forwarders on the Edge VM for specific services:

**SMTP Relay (Systemd Socket Example on Edge VM)**:
```ini
# /etc/systemd/system/smtp-relay.socket
[Unit]
Description=Outbound SMTP Relay Socket for Passly Backend

[Socket]
ListenStream=10.0.0.10:587
Accept=yes

[Install]
WantedBy=sockets.target
```
```ini
# /etc/systemd/system/smtp-relay@.service
[Unit]
Description=Outbound SMTP Relay Connection

[Service]
ExecStart=/usr/bin/systemd-socket-proxyd smtp.email.region.oci.oraclecloud.com:587
```

On the private backend VM, map the upstream SMTP host in `/etc/hosts`:
```text
10.0.0.10  smtp.email.region.oci.oraclecloud.com
```

---

## 5. NixOS on OCI (Optional Alternative)

While standard distributions like Ubuntu or Debian work well, **NixOS** provides distinct operational advantages for cloud deployments, including complete declarative configuration, atomic system generations, instant rollbacks, and zero configuration drift.

For the full operational guide — including `nixos-infect` installation, 1 GB Micro VM memory protection (file-backed swap, single-job constraints, and `systemd-run` resource limits), and declarative Nginx configuration — refer to the dedicated specification:

👉 **[NixOS on OCI: Architecture, Benefits, and Operations](nixos-oci.md)**

---

## 6. Verification and Health Checks

Always verify new deployments using non-destructive, read-only checks:

1. **System Services**:
   ```bash
   systemctl is-active nginx phpfpm-passly mysql
   ```
2. **Passly Internal Health Check**:
   ```bash
   sudo -u www-data bin/cake passbolt healthcheck
   ```
3. **Public HTTP Verification**:
   ```bash
   curl -i https://passly.example.com/healthcheck/status.json
   ```
   *Expected response*: HTTP `200 OK` with JSON body `{"body": "OK"}`.
4. **Session Persistence**:
   - Log in via browser extension or web interface.
   - Restart the PHP-FPM service (`sudo systemctl restart php8.2-fpm`).
   - Refresh the page: the session must remain active without requiring re-authentication.
