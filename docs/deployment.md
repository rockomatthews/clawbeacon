# Deployment Guide

This guide covers deploying Claw Beacon to production using Docker, Railway, and manual server setups.

---

## Table of Contents

- [Deployment Options](#deployment-options)
- [Docker Deployment](#docker-deployment)
- [Railway Deployment](#railway-deployment)
- [Manual Deployment](#manual-deployment)
- [Environment Configuration](#environment-configuration)
- [Reverse Proxy Setup](#reverse-proxy-setup)
- [Monitoring & Maintenance](#monitoring--maintenance)

---

## Deployment Options

| Method | Best For | Complexity | Cost |
|--------|----------|------------|------|
| **Docker Compose** | Self-hosted, full control | Medium | Your infrastructure |
| **Railway** | Quick deployment, managed | Low | ~$5-20/month |
| **Manual** | Custom setups, existing servers | High | Your infrastructure |

---

## Docker Deployment

### Prerequisites

- Docker Engine 20.10+
- Docker Compose v2.0+
- 512MB RAM minimum (1GB recommended)

### Option A: PostgreSQL (Production Recommended)

```bash
# Clone the repository
git clone https://github.com/adarshmishra07/claw-beacon.git
cd claw-beacon

# Create environment file
cat > .env << EOF
POSTGRES_USER=claw
POSTGRES_PASSWORD=$(openssl rand -hex 16)
POSTGRES_DB=claw_beacon
API_KEY=$(openssl rand -hex 32)
EOF

# Start all services
docker-compose up -d

# Check status
docker-compose ps
docker-compose logs -f
```

Access the dashboard at `http://localhost:5173`

### Option B: SQLite (Lightweight)

```bash
# Use the SQLite override
docker-compose -f docker-compose.yml -f docker-compose.sqlite.yml up -d --scale db=0
```

### Docker Compose Configuration

**docker-compose.yml** (PostgreSQL):
```yaml
version: '3.8'

services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: ${POSTGRES_USER:-claw}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-Claw Beacon}
      POSTGRES_DB: ${POSTGRES_DB:-claw_beacon}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-claw}"]
      interval: 5s
      timeout: 5s
      retries: 5

  backend:
    build: ./packages/backend
    environment:
      DATABASE_URL: postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@db:5432/${POSTGRES_DB}
      PORT: 3001
      API_KEY: ${API_KEY}
    volumes:
      - ./config:/app/config:ro
    ports:
      - "3001:3001"
    depends_on:
      db:
        condition: service_healthy
    command: sh -c "npm run migrate && npm start"

  frontend:
    build: ./packages/frontend
    environment:
      VITE_API_URL: http://localhost:3001
    ports:
      - "5173:80"
    depends_on:
      - backend

volumes:
  postgres_data:
```

### Building Custom Images

```bash
# Build with custom tags
docker build -t myregistry/claw-beacon-backend:v1 ./packages/backend
docker build -t myregistry/claw-beacon-frontend:v1 ./packages/frontend

# Push to registry
docker push myregistry/claw-beacon-backend:v1
docker push myregistry/claw-beacon-frontend:v1
```

### Docker Production Tips

1. **Use named volumes** for database persistence
2. **Set resource limits:**
   ```yaml
   backend:
     deploy:
       resources:
         limits:
           memory: 512M
           cpus: '0.5'
   ```
3. **Use secrets** instead of environment variables for sensitive data
4. **Enable health checks** for all services

---

## Railway Deployment

Railway offers one-click deployment with automatic builds and managed PostgreSQL.

### One-Click Deploy

[![Deploy on Railway](https://railway.app/button.svg)](https://railway.app/template/claw-beacon)

### Manual Railway Setup

1. **Create a new project** on [Railway](https://railway.app)

2. **Add PostgreSQL:**
   - Click "New" → "Database" → "PostgreSQL"
   - Railway auto-creates `DATABASE_URL`

3. **Deploy Backend:**
   - Click "New" → "GitHub Repo"
   - Select your Claw Beacon fork
   - Set root directory: `packages/backend`
   - Add environment variables:
     ```
     PORT=3001
     API_KEY=your-secure-key
     ```
   - Railway auto-links `DATABASE_URL` from PostgreSQL

4. **Deploy Frontend:**
   - Click "New" → "GitHub Repo"
   - Select your Claw Beacon fork
   - Set root directory: `packages/frontend`
   - Add environment variables:
     ```
     VITE_API_URL=https://${{@claw-beacon/backend.RAILWAY_PUBLIC_DOMAIN}}
     ```
   - **Note:** The `${{...}}` syntax auto-links to your backend service's URL. Replace `@claw-beacon/backend` with your backend service name if different.

5. **Configure Domains:**
   - Backend: Generate domain or add custom domain
   - Frontend: Generate domain or add custom domain

### Railway Configuration Files

**railway.toml** (root):
```toml
[build]
builder = "nixpacks"

[deploy]
healthcheckPath = "/health"
healthcheckTimeout = 100
restartPolicyType = "on_failure"
restartPolicyMaxRetries = 5
```

**railway.json** (alternative):
```json
{
  "$schema": "https://railway.app/railway.schema.json",
  "build": {
    "builder": "NIXPACKS"
  },
  "deploy": {
    "numReplicas": 1,
    "restartPolicyType": "ON_FAILURE",
    "restartPolicyMaxRetries": 10
  }
}
```

### Railway Costs

- **Hobby Plan:** ~$5/month for light usage
- **Pro Plan:** ~$20/month for production workloads
- PostgreSQL included in pricing

---

## Manual Deployment

For deploying on your own VPS or bare metal server.

### Prerequisites

- Ubuntu 22.04 LTS (or similar)
- Node.js 18+
- PostgreSQL 14+ (or use SQLite)
- Nginx (for reverse proxy)
- PM2 or systemd (for process management)

### Step 1: Install Dependencies

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Node.js 18
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs

# Install PostgreSQL
sudo apt install -y postgresql postgresql-contrib

# Install Nginx
sudo apt install -y nginx
```

### Step 2: Setup PostgreSQL

```bash
# Create database and user
sudo -u postgres psql << EOF
CREATE USER claw WITH PASSWORD 'your-secure-password';
CREATE DATABASE claw_beacon OWNER claw;
GRANT ALL PRIVILEGES ON DATABASE claw_beacon TO claw;
EOF
```

### Step 3: Deploy Application

```bash
# Create app directory
sudo mkdir -p /opt/claw-beacon
sudo chown $USER:$USER /opt/claw-beacon

# Clone repository
git clone https://github.com/adarshmishra07/claw-beacon.git /opt/claw-beacon
cd /opt/claw-beacon

# Setup backend
cd packages/backend
npm install --production

cat > .env << EOF
DATABASE_URL=postgresql://claw:your-secure-password@localhost:5432/claw_beacon
PORT=3001
API_KEY=$(openssl rand -hex 32)
EOF

npm run migrate

# Setup frontend
cd ../frontend
npm install
npm run build
```

### Step 4: Process Management with PM2

```bash
# Install PM2
sudo npm install -g pm2

# Start backend
cd /opt/claw-beacon/packages/backend
pm2 start src/index.js --name claw-backend

# Save PM2 config
pm2 save
pm2 startup
```

### Step 5: Systemd Service (Alternative to PM2)

Create `/etc/systemd/system/claw-beacon.service`:

```ini
[Unit]
Description=Claw Beacon Backend
After=network.target postgresql.service

[Service]
Type=simple
User=www-data
WorkingDirectory=/opt/claw-beacon/packages/backend
ExecStart=/usr/bin/node src/index.js
Restart=on-failure
RestartSec=10
Environment=NODE_ENV=production

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable claw-beacon
sudo systemctl start claw-beacon
sudo systemctl status claw-beacon
```

---

## Environment Configuration

### Production Environment Variables

**Backend (.env):**
```env
# Database
DATABASE_URL=postgresql://user:password@localhost:5432/claw_beacon

# Server
PORT=3001
NODE_ENV=production

# Security
API_KEY=your-32-character-secure-key

# Optional: Custom config paths
AGENTS_CONFIG_PATH=/opt/claw-beacon/config/agents.yaml
WEBHOOKS_CONFIG_PATH=/opt/claw-beacon/config/webhooks.yaml
```

**Frontend (.env):**
```env
VITE_API_URL=https://api.yourdomain.com
```

### Generating Secure Keys

```bash
# Generate API key
openssl rand -hex 32

# Generate PostgreSQL password
openssl rand -base64 24
```

---

## Reverse Proxy Setup

### Nginx Configuration

Create `/etc/nginx/sites-available/claw-beacon`:

```nginx
# Backend API
server {
    listen 80;
    server_name api.yourdomain.com;

    location / {
        proxy_pass http://127.0.0.1:3001;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }

    # SSE specific settings
    location /api/stream {
        proxy_pass http://127.0.0.1:3001;
        proxy_http_version 1.1;
        proxy_set_header Connection '';
        proxy_buffering off;
        proxy_cache off;
        proxy_read_timeout 24h;
    }
}

# Frontend
server {
    listen 80;
    server_name yourdomain.com;

    root /opt/claw-beacon/packages/frontend/dist;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    # Cache static assets
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

Enable the site:

```bash
sudo ln -s /etc/nginx/sites-available/claw-beacon /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

### SSL with Let's Encrypt

```bash
# Install Certbot
sudo apt install certbot python3-certbot-nginx

# Get certificates
sudo certbot --nginx -d yourdomain.com -d api.yourdomain.com

# Auto-renewal is configured automatically
sudo certbot renew --dry-run
```

---

## Monitoring & Maintenance

### Health Checks

Monitor the `/health` endpoint:

```bash
# Simple check
curl -s http://localhost:3001/health | jq

# With alerting (using curl + webhook)
if ! curl -sf http://localhost:3001/health > /dev/null; then
  curl -X POST https://your-alert-webhook.com -d "Claw Beacon is down!"
fi
```

### Logs

**PM2 logs:**
```bash
pm2 logs claw-backend
pm2 logs claw-backend --lines 100
```

**Systemd logs:**
```bash
sudo journalctl -u claw-beacon -f
sudo journalctl -u claw-beacon --since "1 hour ago"
```

### Database Backups

**PostgreSQL:**
```bash
# Backup
pg_dump -U claw claw_beacon > backup_$(date +%Y%m%d).sql

# Restore
psql -U claw claw_beacon < backup_20240115.sql
```

**SQLite:**
```bash
# Backup (just copy the file)
cp data/claw-beacon.db backups/claw-beacon_$(date +%Y%m%d).db
```

### Automated Backup Script

Create `/opt/claw-beacon/backup.sh`:

```bash
#!/bin/bash
BACKUP_DIR=/opt/backups/claw-beacon
DATE=$(date +%Y%m%d_%H%M%S)

mkdir -p $BACKUP_DIR

# Backup database
pg_dump -U claw claw_beacon | gzip > $BACKUP_DIR/db_$DATE.sql.gz

# Backup config
tar -czf $BACKUP_DIR/config_$DATE.tar.gz /opt/claw-beacon/config/

# Keep only last 7 days
find $BACKUP_DIR -type f -mtime +7 -delete

echo "Backup completed: $DATE"
```

Add to cron:
```bash
# Daily at 2 AM
0 2 * * * /opt/claw-beacon/backup.sh >> /var/log/claw-backup.log 2>&1
```

### Updates

```bash
cd /opt/claw-beacon

# Pull latest
git pull origin main

# Update backend
cd packages/backend
npm install
npm run migrate

# Update frontend
cd ../frontend
npm install
npm run build

# Restart services
pm2 restart claw-backend
# or
sudo systemctl restart claw-beacon
```

---

## Security Checklist

Before going live, ensure:

- [ ] **API_KEY is set** - Protects write operations
- [ ] **SSL/TLS enabled** - Use HTTPS everywhere
- [ ] **Database secured** - Strong passwords, no public access
- [ ] **Firewall configured** - Only expose necessary ports
- [ ] **Backups configured** - Regular automated backups
- [ ] **Monitoring active** - Health checks and alerts
- [ ] **Logs rotated** - Prevent disk filling up
- [ ] **Updates planned** - Regular security patches

---

<p align="center">
  You're deployed! 🎉 Questions? Check <a href="https://github.com/adarshmishra07/claw-beacon/issues">GitHub Issues</a> 🦞
</p>
