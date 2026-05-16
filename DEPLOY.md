# Plan: Deploy TaskChampion Sync Server to VPS

## Goal
Deploy taskchampion-sync-server-postgres to VPS with PostgreSQL backend, nginx reverse proxy with HTTPS, and systemd service management.

## Architecture
```
Internet -> nginx (443/HTTPS) -> taskchampion-sync-server (127.0.0.1:8080) -> PostgreSQL (local)
```

---

## Step 1: PostgreSQL Setup on VPS

**Clarification**: `pg_hba.conf` controls *authentication rules* (who can connect and how). Creating the database and user is done with SQL commands.

```bash
# Connect as postgres superuser
sudo -u postgres psql

# Create user and database (in psql)
CREATE USER taskchampion WITH PASSWORD 'your-secure-password-here';
CREATE DATABASE taskchampion OWNER taskchampion;
\q
```

**pg_hba.conf** (usually `/etc/postgresql/*/main/pg_hba.conf`) - ensure local connections work:
```
# TYPE  DATABASE        USER            ADDRESS         METHOD
local   taskchampion    taskchampion                    scram-sha-256
host    taskchampion    taskchampion    127.0.0.1/32    scram-sha-256
```

Then reload: `sudo systemctl reload postgresql`

---

## Step 2: Domain DNS

Point your domain (e.g., `taskchampion.yourdomain.com`) A record to your VPS IP.

---

## Step 3: Deploy Binary

```bash
# From local machine
scp /home/ryan/repos/ups/builds/taskchampion-sync-server/taskchampion-sync-server-postgres user@vps:/usr/local/bin/

# On VPS - set permissions
sudo chmod +x /usr/local/bin/taskchampion-sync-server-postgres
```

---

## Step 4: Systemd Service

Create `/etc/systemd/system/taskchampion-sync-server.service`:

```ini
[Unit]
Description=TaskChampion Sync Server
After=network.target postgresql.service
Requires=postgresql.service

[Service]
Type=simple
User=taskchampion
Group=taskchampion
ExecStart=/usr/local/bin/taskchampion-sync-server-postgres \
    --listen 127.0.0.1:8080 \
    --connection "postgresql://taskchampion:PASSWORD@localhost/taskchampion" \
    --no-create-clients \
    --allow-client-id YOUR-CLIENT-UUID-HERE
Restart=on-failure
RestartSec=5

# Security hardening
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

**Note**: Generate client UUID with `uuidgen` - this is what you'll configure in taskwarrior.

```bash
# Create service user
sudo useradd --system --no-create-home taskchampion

# Enable and start
sudo systemctl daemon-reload
sudo systemctl enable taskchampion-sync-server
sudo systemctl start taskchampion-sync-server
```

---

## Step 5: Nginx Reverse Proxy

Create `/etc/nginx/sites-available/taskchampion`:

```nginx
server {
    listen 80;
    server_name taskchampion.yourdomain.com;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/taskchampion /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

---

## Step 6: HTTPS with Certbot

```bash
sudo certbot --nginx -d taskchampion.yourdomain.com
```

Certbot will automatically modify the nginx config to add SSL.

---

## Step 7: Configure Taskwarrior Client

In `~/.taskrc` on your local machine:
```
sync.server.url=https://taskchampion.yourdomain.com
sync.server.client_id=YOUR-CLIENT-UUID-HERE
sync.server.encryption_secret=your-encryption-secret
```

---

## Verification

1. Check service status: `sudo systemctl status taskchampion-sync-server`
2. Check logs: `sudo journalctl -u taskchampion-sync-server -f`
3. Test HTTPS endpoint: `curl https://taskchampion.yourdomain.com/` (expect 404 or similar - just verifying connectivity)
4. Test sync from client: `task sync`

---

## Security Notes

- The server listens only on 127.0.0.1 (not exposed directly)
- `--no-create-clients` prevents unauthorized clients
- `--allow-client-id` whitelist restricts to known UUIDs
- HTTPS terminates at nginx
- PostgreSQL only accepts local connections
