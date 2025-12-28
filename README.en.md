# Portainer Docker Stack with Traefik Integration

[🇩🇪 Deutsch](README.md) | [🇫🇷 Français](README.fr.md)

A production-ready Docker stack for Portainer CE with full Traefik v2+ integration, HTTPS/SSL support, and modern security features.

## 📋 Table of Contents

- [Features](#-features)
- [Prerequisites](#-prerequisites)
- [Installation](#-installation)
- [Configuration](#-configuration)
- [Traefik Labels](#-traefik-labels)
- [Usage](#-usage)
- [Security](#-security)
- [Troubleshooting](#-troubleshooting)
- [License](#-license)

## ✨ Features

- ✅ **Portainer CE** - Modern Docker management interface
- ✅ **Traefik Integration** - Automatic routing and SSL certificates
- ✅ **Let's Encrypt** - Automatic HTTPS certificates
- ✅ **Persistent Storage** - Data persists across container updates
- ✅ **Production-Ready** - Best practices for security and performance

## 🔧 Prerequisites

- Docker Engine 20.10+
- Docker Compose v2.0+
- Running Traefik v2+ reverse proxy
- Domain with DNS record pointing to your server

### Traefik Network

The stack requires an external Docker network for Traefik:

```bash
docker network create traefik_proxy_network
```

### Required Traefik Middlewares

This stack uses the following Traefik middlewares (optional):
- `redirect-to-https@file` - HTTP → HTTPS redirect
- `geo-block@file` - Geo-blocking (23 high-risk countries)
- `security-headers@file` - Security headers (HSTS, X-Frame-Options)
- `compression@file` - Gzip/Brotli compression
- `rate-limit@file` - DoS protection (100 req/s)

If these middlewares are not available, remove them from the labels or create them in your `traefik-dynamic.yaml`.

## 📦 Installation

### Step 1: Clone repository

```bash
git clone https://github.com/csaeum/DockerStackPortainer.git
cd DockerStackPortainer
```

### Step 2: Configure environment variables

```bash
cp .env.example .env
nano .env
```

Adjust the values in `.env`:

```env
COMPOSE_PROJECT_NAME=portainer
HOSTRULE=Host(`portainer.your-domain.com`)
PROXY_NETWORK=traefik_proxy_network
RESTART=unless-stopped
```

### Step 3: Start the stack

```bash
docker-compose up -d
```

### Step 4: Initial setup

1. Open `https://portainer.your-domain.com` in your browser
2. Create an admin user (password min. 12 characters)
3. Select "Docker Standalone" as environment
4. Done! You can now manage Docker containers

## ⚙️ Configuration

### docker-compose.yaml

The stack uses the following configuration:

```yaml
volumes:
  portainer_data:
    driver: local
    driver_opts:
      device: ${PWD}/volumes  # Persistent storage
      o: bind
      type: none

services:
  portainer:
    image: portainer/portainer-ce:lts
    command: -H unix:///var/run/docker.sock
    container_name: ${COMPOSE_PROJECT_NAME}
    networks:
      - ${PROXY_NETWORK}
    restart: ${RESTART}
    volumes:
      - portainer_data:/data
      - /var/run/docker.sock:/var/run/docker.sock  # Docker access
```

### Volumes

Portainer data is stored in `./volumes/`:
- Users, teams, roles
- Settings and configurations
- Container templates
- Registries

**Important:** Back up this folder regularly!

## 🏷️ Traefik Labels

### Basic Configuration (current)

```yaml
labels:
  - traefik.enable=true
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}.rule=${HOSTRULE}
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}.entrypoints=websecure-https
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}.tls.certresolver=letsEncrypt
  - traefik.http.services.${COMPOSE_PROJECT_NAME}.loadBalancer.server.port=9000
```

### Advanced Configuration (recommended)

For maximum security, add the following labels:

```yaml
labels:
  - traefik.enable=true

  # HTTP Router (for HTTPS redirect)
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}-http.rule=${HOSTRULE}
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}-http.entrypoints=web-http
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}-http.middlewares=redirect-to-https@file

  # HTTPS Router
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}.rule=${HOSTRULE}
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}.entrypoints=websecure-https
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}.tls.certresolver=letsEncrypt
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}.tls.options=modern@file
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}.middlewares=geo-block@file,security-headers@file,compression@file,rate-limit@file
  - traefik.http.services.${COMPOSE_PROJECT_NAME}.loadBalancer.server.port=9000
```

**Benefits:**
- ✅ Automatic HTTP → HTTPS redirect
- ✅ TLS 1.3 enforced
- ✅ Geo-blocking against high-risk countries
- ✅ HSTS and security headers
- ✅ Brute-force protection via rate-limiting

## 🚀 Usage

### Stack Management

```bash
# Start stack
docker-compose up -d

# View logs
docker-compose logs -f

# Stop stack
docker-compose down

# Restart stack
docker-compose restart

# Delete stack including volumes (CAUTION!)
docker-compose down -v
```

### Create Backup

```bash
# Stop Portainer
docker-compose down

# Create backup
tar -czf portainer-backup-$(date +%Y%m%d).tar.gz volumes/

# Restart stack
docker-compose up -d
```

### Restore from Backup

```bash
# Stop stack
docker-compose down

# Delete old data
rm -rf volumes/*

# Restore backup
tar -xzf portainer-backup-YYYYMMDD.tar.gz

# Start stack
docker-compose up -d
```

## 🔒 Security

### Best Practices

1. **Strong Admin Password** - Min. 16 characters, special chars, numbers
2. **Enable 2FA** - In Portainer: User → My account → Enable 2FA
3. **Use Geo-Blocking** - Traefik middleware `geo-block@file`
4. **Rate-Limiting** - Protection against brute-force attacks
5. **Regular Updates** - `docker-compose pull && docker-compose up -d`
6. **IP Whitelisting** - If you always access from the same IP

### IP Whitelisting (optional)

Create a Traefik middleware:

```yaml
# In traefik-dynamic.yaml
http:
  middlewares:
    portainer-ip-whitelist:
      ipWhiteList:
        sourceRange:
          - "192.168.1.0/24"  # Your local network
          - "1.2.3.4/32"       # Your office IP
```

Add it to the labels:

```yaml
- traefik.http.routers.${COMPOSE_PROJECT_NAME}.middlewares=portainer-ip-whitelist@file,geo-block@file,security-headers@file
```

### Docker Socket Security

Portainer requires access to `/var/run/docker.sock` - this means **full root access** to the Docker host!

**Important:**
- Never expose Portainer publicly without authentication
- Only grant trusted users Portainer admin access
- Regularly review audit logs

## 🐛 Troubleshooting

### Problem: 404 Not Found

**Cause:** Container not in Traefik network

```bash
# Check network
docker network inspect traefik_proxy_network

# Restart container
docker-compose down && docker-compose up -d
```

### Problem: 502 Bad Gateway

**Cause:** Port 9000 is incorrect or container not running

```bash
# Check container status
docker-compose ps

# Check logs
docker-compose logs portainer

# Inspect container network
docker exec -it portainer sh
```

### Problem: Let's Encrypt Certificate Error

**Cause:** DNS not pointing to your server or rate-limit reached

```bash
# Check DNS
dig portainer.your-domain.com +short

# Check Traefik logs
docker logs traefik 2>&1 | grep -i acme
```

### Problem: 401 Unauthorized after Update

**Cause:** Session cookies invalid after container restart

```bash
# Clear browser cache
# Or: Use private/incognito window
```

## 📊 Stack Information

- **Portainer Version:** `portainer/portainer-ce:lts`
- **Required Port:** 9000 (internal only, accessible via Traefik)
- **Volumes:** `./volumes/` (bind-mount)
- **Network:** `traefik_proxy_network` (external)

## 🤝 Contributing

Contributions are welcome! Please create a pull request or open an issue.

## 📄 License

This project is licensed under the GPL-3.0-or-later License - see [LICENSE](LICENSE) for details.

---

**Made with ❤️ by WSC - Web SEO Consulting**

This project is free and open source. If it helped you, I appreciate your support:

[![Buy Me a Coffee](https://img.shields.io/badge/Buy%20Me%20a%20Coffee-ffdd00?style=for-the-badge&logo=buy-me-a-coffee&logoColor=black)](https://buymeacoffee.com/csaeum)
[![GitHub Sponsors](https://img.shields.io/badge/GitHub%20Sponsors-ea4aaa?style=for-the-badge&logo=github-sponsors&logoColor=white)](https://github.com/sponsors/csaeum)
[![PayPal](https://img.shields.io/badge/PayPal-00457C?style=for-the-badge&logo=paypal&logoColor=white)](https://paypal.me/csaeum)

---

## 🔗 Links

- [Portainer Documentation](https://docs.portainer.io/)
- [Traefik Documentation](https://doc.traefik.io/traefik/)
- [Docker Documentation](https://docs.docker.com/)
