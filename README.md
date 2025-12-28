# Portainer Docker Stack mit Traefik Integration

[🇬🇧 English](README.en.md) | [🇫🇷 Français](README.fr.md)

Ein production-ready Docker-Stack für Portainer CE mit vollständiger Traefik v2+ Integration, HTTPS/SSL-Unterstützung und modernen Security-Features.

## 📋 Inhaltsverzeichnis

- [Funktionen](#-funktionen)
- [Voraussetzungen](#-voraussetzungen)
- [Installation](#-installation)
- [Konfiguration](#-konfiguration)
- [Traefik Labels](#-traefik-labels)
- [Nutzung](#-nutzung)
- [Sicherheit](#-sicherheit)
- [Troubleshooting](#-troubleshooting)
- [Lizenz](#-lizenz)

## ✨ Funktionen

- ✅ **Portainer CE** - Modernes Docker-Management-Interface
- ✅ **Traefik Integration** - Automatisches Routing und SSL-Zertifikate
- ✅ **Let's Encrypt** - Automatische HTTPS-Zertifikate
- ✅ **Persistent Storage** - Daten bleiben bei Container-Updates erhalten
- ✅ **Production-Ready** - Best Practices für Sicherheit und Performance

## 🔧 Voraussetzungen

- Docker Engine 20.10+
- Docker Compose v2.0+
- Laufender Traefik v2+ Reverse Proxy
- Domain mit DNS-Eintrag auf deinen Server

### Traefik Network

Der Stack benötigt ein externes Docker-Network für Traefik:

```bash
docker network create traefik_proxy_network
```

### Erforderliche Traefik Middlewares

Dieser Stack nutzt folgende Traefik-Middlewares (optional):
- `redirect-to-https@file` - HTTP → HTTPS Weiterleitung
- `geo-block@file` - Geo-Blocking (23 Risiko-Länder)
- `security-headers@file` - Security Headers (HSTS, X-Frame-Options)
- `compression@file` - Gzip/Brotli Kompression
- `rate-limit@file` - DoS-Schutz (100 req/s)

Wenn diese Middlewares nicht vorhanden sind, entferne sie aus den Labels oder erstelle sie in deiner `traefik-dynamic.yaml`.

## 📦 Installation

### Schritt 1: Repository klonen

```bash
git clone https://github.com/csaeum/DockerStackPortainer.git
cd DockerStackPortainer
```

### Schritt 2: Umgebungsvariablen konfigurieren

```bash
cp .env.example .env
nano .env
```

Passe die Werte in `.env` an:

```env
COMPOSE_PROJECT_NAME=portainer
HOSTRULE=Host(`portainer.deine-domain.de`)
PROXY_NETWORK=traefik_proxy_network
RESTART=unless-stopped
```

### Schritt 3: Stack starten

```bash
docker-compose up -d
```

### Schritt 4: Initial Setup

1. Öffne `https://portainer.deine-domain.de` im Browser
2. Erstelle einen Admin-Benutzer (Passwort mind. 12 Zeichen)
3. Wähle "Docker Standalone" als Environment
4. Fertig! Du kannst jetzt Docker-Container verwalten

## ⚙️ Konfiguration

### docker-compose.yaml

Der Stack nutzt folgende Konfiguration:

```yaml
volumes:
  portainer_data:
    driver: local
    driver_opts:
      device: ${PWD}/volumes  # Persistenter Speicher
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
      - /var/run/docker.sock:/var/run/docker.sock  # Docker-Zugriff
```

### Volumes

Portainer-Daten werden in `./volumes/` gespeichert:
- Benutzer, Teams, Rollen
- Einstellungen und Konfigurationen
- Container-Templates
- Registries

**Wichtig:** Sichere diesen Ordner regelmäßig!

## 🏷️ Traefik Labels

### Basis-Konfiguration (aktuell)

```yaml
labels:
  - traefik.enable=true
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}.rule=${HOSTRULE}
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}.entrypoints=websecure-https
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}.tls.certresolver=letsEncrypt
  - traefik.http.services.${COMPOSE_PROJECT_NAME}.loadBalancer.server.port=9000
```

### Erweiterte Konfiguration (empfohlen)

Für maximale Sicherheit füge folgende Labels hinzu:

```yaml
labels:
  - traefik.enable=true

  # HTTP Router (für HTTPS-Redirect)
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

**Vorteile:**
- ✅ Automatischer HTTP → HTTPS Redirect
- ✅ TLS 1.3 erzwungen
- ✅ Geo-Blocking gegen Risiko-Länder
- ✅ HSTS und Security Headers
- ✅ Brute-Force Schutz durch Rate-Limiting

## 🚀 Nutzung

### Stack-Verwaltung

```bash
# Stack starten
docker-compose up -d

# Logs anzeigen
docker-compose logs -f

# Stack stoppen
docker-compose down

# Stack neu starten
docker-compose restart

# Stack inkl. Volumes löschen (VORSICHT!)
docker-compose down -v
```

### Backup erstellen

```bash
# Portainer stoppen
docker-compose down

# Backup erstellen
tar -czf portainer-backup-$(date +%Y%m%d).tar.gz volumes/

# Stack wieder starten
docker-compose up -d
```

### Restore aus Backup

```bash
# Stack stoppen
docker-compose down

# Alte Daten löschen
rm -rf volumes/*

# Backup wiederherstellen
tar -xzf portainer-backup-YYYYMMDD.tar.gz

# Stack starten
docker-compose up -d
```

## 🔒 Sicherheit

### Best Practices

1. **Starkes Admin-Passwort** - Mind. 16 Zeichen, Sonderzeichen, Zahlen
2. **2FA aktivieren** - In Portainer: User → My account → Enable 2FA
3. **Geo-Blocking nutzen** - Traefik Middleware `geo-block@file`
4. **Rate-Limiting** - Schutz vor Brute-Force-Angriffen
5. **Regelmäßige Updates** - `docker-compose pull && docker-compose up -d`
6. **IP-Whitelisting** - Falls du immer von der gleichen IP zugreifst

### IP-Whitelisting (optional)

Erstelle eine Traefik-Middleware:

```yaml
# In traefik-dynamic.yaml
http:
  middlewares:
    portainer-ip-whitelist:
      ipWhiteList:
        sourceRange:
          - "192.168.1.0/24"  # Dein lokales Netzwerk
          - "1.2.3.4/32"       # Deine Office-IP
```

Füge sie zu den Labels hinzu:

```yaml
- traefik.http.routers.${COMPOSE_PROJECT_NAME}.middlewares=portainer-ip-whitelist@file,geo-block@file,security-headers@file
```

### Docker Socket Security

Portainer benötigt Zugriff auf `/var/run/docker.sock` - das bedeutet **voller Root-Zugriff** auf den Docker-Host!

**Wichtig:**
- Niemals Portainer öffentlich ohne Auth zugänglich machen
- Nur vertrauenswürdige Benutzer als Portainer-Admins
- Regelmäßig Audit-Logs prüfen

## 🐛 Troubleshooting

### Problem: 404 Not Found

**Ursache:** Container nicht im Traefik-Network

```bash
# Network prüfen
docker network inspect traefik_proxy_network

# Container neu starten
docker-compose down && docker-compose up -d
```

### Problem: 502 Bad Gateway

**Ursache:** Port 9000 ist falsch oder Container läuft nicht

```bash
# Container-Status prüfen
docker-compose ps

# Logs prüfen
docker-compose logs portainer

# Ins Container-Netzwerk schauen
docker exec -it portainer sh
```

### Problem: Let's Encrypt Zertifikat-Fehler

**Ursache:** DNS zeigt nicht auf deinen Server oder Rate-Limit erreicht

```bash
# DNS prüfen
dig portainer.deine-domain.de +short

# Traefik-Logs prüfen
docker logs traefik 2>&1 | grep -i acme
```

### Problem: 401 Unauthorized nach Update

**Ursache:** Session-Cookies ungültig nach Container-Neustart

```bash
# Browser-Cache löschen
# Oder: Private/Incognito-Fenster nutzen
```

## 📊 Stack-Informationen

- **Portainer Version:** `portainer/portainer-ce:lts`
- **Benötigter Port:** 9000 (nur intern, über Traefik erreichbar)
- **Volumes:** `./volumes/` (bind-mount)
- **Network:** `traefik_proxy_network` (external)

## 🤝 Beitragen

Contributions sind willkommen! Bitte erstelle einen Pull Request oder öffne ein Issue.

## 📄 Lizenz

Dieses Projekt steht unter der GPL-3.0-or-later Lizenz - siehe [LICENSE](LICENSE) für Details.

---

**Made with ❤️ by WSC - Web SEO Consulting**

Dieses Projekt ist kostenlos und Open Source. Wenn es dir geholfen hat, freue ich mich über deine Unterstützung:

[![Buy Me a Coffee](https://img.shields.io/badge/Buy%20Me%20a%20Coffee-ffdd00?style=for-the-badge&logo=buy-me-a-coffee&logoColor=black)](https://buymeacoffee.com/csaeum)
[![GitHub Sponsors](https://img.shields.io/badge/GitHub%20Sponsors-ea4aaa?style=for-the-badge&logo=github-sponsors&logoColor=white)](https://github.com/sponsors/csaeum)
[![PayPal](https://img.shields.io/badge/PayPal-00457C?style=for-the-badge&logo=paypal&logoColor=white)](https://paypal.me/csaeum)

---

## 🔗 Links

- [Portainer Documentation](https://docs.portainer.io/)
- [Traefik Documentation](https://doc.traefik.io/traefik/)
- [Docker Documentation](https://docs.docker.com/)
