# Stack Docker Portainer avec intégration Traefik

[🇩🇪 Deutsch](README.md) | [🇬🇧 English](README.en.md)

Un stack Docker prêt pour la production pour Portainer CE avec une intégration complète de Traefik v2+, support HTTPS/SSL et fonctionnalités de sécurité modernes.

## 📋 Table des matières

- [Fonctionnalités](#-fonctionnalités)
- [Prérequis](#-prérequis)
- [Installation](#-installation)
- [Configuration](#-configuration)
- [Labels Traefik](#-labels-traefik)
- [Utilisation](#-utilisation)
- [Sécurité](#-sécurité)
- [Dépannage](#-dépannage)
- [Licence](#-licence)

## ✨ Fonctionnalités

- ✅ **Portainer CE** - Interface moderne de gestion Docker
- ✅ **Intégration Traefik** - Routage automatique et certificats SSL
- ✅ **Let's Encrypt** - Certificats HTTPS automatiques
- ✅ **Stockage persistant** - Les données persistent lors des mises à jour
- ✅ **Prêt pour production** - Bonnes pratiques pour sécurité et performance

## 🔧 Prérequis

- Docker Engine 20.10+
- Docker Compose v2.0+
- Proxy inverse Traefik v2+ en cours d'exécution
- Domaine avec enregistrement DNS pointant vers votre serveur

### Réseau Traefik

Le stack nécessite un réseau Docker externe pour Traefik :

```bash
docker network create traefik_proxy_network
```

### Middlewares Traefik requis

Ce stack utilise les middlewares Traefik suivants (optionnel) :
- `redirect-to-https@file` - Redirection HTTP → HTTPS
- `geo-block@file` - Géo-blocage (23 pays à haut risque)
- `security-headers@file` - En-têtes de sécurité (HSTS, X-Frame-Options)
- `compression@file` - Compression Gzip/Brotli
- `rate-limit@file` - Protection DoS (100 req/s)

Si ces middlewares ne sont pas disponibles, supprimez-les des labels ou créez-les dans votre `traefik-dynamic.yaml`.

## 📦 Installation

### Étape 1 : Cloner le dépôt

```bash
git clone https://github.com/csaeum/DockerStackPortainer.git
cd DockerStackPortainer
```

### Étape 2 : Configurer les variables d'environnement

```bash
cp .env.example .env
nano .env
```

Ajustez les valeurs dans `.env` :

```env
COMPOSE_PROJECT_NAME=portainer
HOSTRULE=Host(`portainer.votre-domaine.fr`)
PROXY_NETWORK=traefik_proxy_network
RESTART=unless-stopped
```

### Étape 3 : Démarrer le stack

```bash
docker-compose up -d
```

### Étape 4 : Configuration initiale

1. Ouvrez `https://portainer.votre-domaine.fr` dans votre navigateur
2. Créez un utilisateur admin (mot de passe min. 12 caractères)
3. Sélectionnez "Docker Standalone" comme environnement
4. Terminé ! Vous pouvez maintenant gérer les conteneurs Docker

## ⚙️ Configuration

### docker-compose.yaml

Le stack utilise la configuration suivante :

```yaml
volumes:
  portainer_data:
    driver: local
    driver_opts:
      device: ${PWD}/volumes  # Stockage persistant
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
      - /var/run/docker.sock:/var/run/docker.sock  # Accès Docker
```

### Volumes

Les données Portainer sont stockées dans `./volumes/` :
- Utilisateurs, équipes, rôles
- Paramètres et configurations
- Modèles de conteneurs
- Registres

**Important :** Sauvegardez régulièrement ce dossier !

## 🏷️ Labels Traefik

### Configuration de base (actuelle)

```yaml
labels:
  - traefik.enable=true
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}.rule=${HOSTRULE}
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}.entrypoints=websecure-https
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}.tls.certresolver=letsEncrypt
  - traefik.http.services.${COMPOSE_PROJECT_NAME}.loadBalancer.server.port=9000
```

### Configuration avancée (recommandée)

Pour une sécurité maximale, ajoutez les labels suivants :

```yaml
labels:
  - traefik.enable=true

  # Routeur HTTP (pour redirection HTTPS)
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}-http.rule=${HOSTRULE}
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}-http.entrypoints=web-http
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}-http.middlewares=redirect-to-https@file

  # Routeur HTTPS
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}.rule=${HOSTRULE}
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}.entrypoints=websecure-https
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}.tls.certresolver=letsEncrypt
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}.tls.options=modern@file
  - traefik.http.routers.${COMPOSE_PROJECT_NAME}.middlewares=geo-block@file,security-headers@file,compression@file,rate-limit@file
  - traefik.http.services.${COMPOSE_PROJECT_NAME}.loadBalancer.server.port=9000
```

**Avantages :**
- ✅ Redirection automatique HTTP → HTTPS
- ✅ TLS 1.3 forcé
- ✅ Géo-blocage contre les pays à haut risque
- ✅ HSTS et en-têtes de sécurité
- ✅ Protection contre force brute via rate-limiting

## 🚀 Utilisation

### Gestion du stack

```bash
# Démarrer le stack
docker-compose up -d

# Voir les logs
docker-compose logs -f

# Arrêter le stack
docker-compose down

# Redémarrer le stack
docker-compose restart

# Supprimer le stack avec volumes (ATTENTION !)
docker-compose down -v
```

### Créer une sauvegarde

```bash
# Arrêter Portainer
docker-compose down

# Créer la sauvegarde
tar -czf portainer-backup-$(date +%Y%m%d).tar.gz volumes/

# Redémarrer le stack
docker-compose up -d
```

### Restaurer depuis une sauvegarde

```bash
# Arrêter le stack
docker-compose down

# Supprimer anciennes données
rm -rf volumes/*

# Restaurer la sauvegarde
tar -xzf portainer-backup-YYYYMMDD.tar.gz

# Démarrer le stack
docker-compose up -d
```

## 🔒 Sécurité

### Bonnes pratiques

1. **Mot de passe admin fort** - Min. 16 caractères, caractères spéciaux, chiffres
2. **Activer 2FA** - Dans Portainer : User → My account → Enable 2FA
3. **Utiliser le géo-blocage** - Middleware Traefik `geo-block@file`
4. **Rate-limiting** - Protection contre les attaques par force brute
5. **Mises à jour régulières** - `docker-compose pull && docker-compose up -d`
6. **Liste blanche IP** - Si vous accédez toujours depuis la même IP

### Liste blanche IP (optionnel)

Créez un middleware Traefik :

```yaml
# Dans traefik-dynamic.yaml
http:
  middlewares:
    portainer-ip-whitelist:
      ipWhiteList:
        sourceRange:
          - "192.168.1.0/24"  # Votre réseau local
          - "1.2.3.4/32"       # Votre IP bureau
```

Ajoutez-le aux labels :

```yaml
- traefik.http.routers.${COMPOSE_PROJECT_NAME}.middlewares=portainer-ip-whitelist@file,geo-block@file,security-headers@file
```

### Sécurité du socket Docker

Portainer nécessite l'accès à `/var/run/docker.sock` - cela signifie **accès root complet** à l'hôte Docker !

**Important :**
- Ne jamais exposer Portainer publiquement sans authentification
- Accorder l'accès admin Portainer uniquement aux utilisateurs de confiance
- Vérifier régulièrement les journaux d'audit

## 🐛 Dépannage

### Problème : 404 Not Found

**Cause :** Conteneur pas dans le réseau Traefik

```bash
# Vérifier le réseau
docker network inspect traefik_proxy_network

# Redémarrer le conteneur
docker-compose down && docker-compose up -d
```

### Problème : 502 Bad Gateway

**Cause :** Port 9000 incorrect ou conteneur ne fonctionne pas

```bash
# Vérifier le statut du conteneur
docker-compose ps

# Vérifier les logs
docker-compose logs portainer

# Inspecter le réseau du conteneur
docker exec -it portainer sh
```

### Problème : Erreur de certificat Let's Encrypt

**Cause :** DNS ne pointe pas vers votre serveur ou limite de taux atteinte

```bash
# Vérifier DNS
dig portainer.votre-domaine.fr +short

# Vérifier les logs Traefik
docker logs traefik 2>&1 | grep -i acme
```

### Problème : 401 Unauthorized après mise à jour

**Cause :** Cookies de session invalides après redémarrage du conteneur

```bash
# Vider le cache du navigateur
# Ou : Utiliser une fenêtre privée/incognito
```

## 📊 Informations sur le stack

- **Version Portainer :** `portainer/portainer-ce:lts`
- **Port requis :** 9000 (interne uniquement, accessible via Traefik)
- **Volumes :** `./volumes/` (bind-mount)
- **Réseau :** `traefik_proxy_network` (externe)

## 🤝 Contribuer

Les contributions sont les bienvenues ! Veuillez créer une pull request ou ouvrir une issue.

## 📄 Licence

Ce projet est sous licence GPL-3.0-or-later - voir [LICENSE](LICENSE) pour plus de détails.

---

**Fait avec ❤️ par WSC - Web SEO Consulting**

Ce projet est gratuit et open source. S'il vous a aidé, j'apprécie votre soutien :

[![Buy Me a Coffee](https://img.shields.io/badge/Buy%20Me%20a%20Coffee-ffdd00?style=for-the-badge&logo=buy-me-a-coffee&logoColor=black)](https://buymeacoffee.com/csaeum)
[![GitHub Sponsors](https://img.shields.io/badge/GitHub%20Sponsors-ea4aaa?style=for-the-badge&logo=github-sponsors&logoColor=white)](https://github.com/sponsors/csaeum)
[![PayPal](https://img.shields.io/badge/PayPal-00457C?style=for-the-badge&logo=paypal&logoColor=white)](https://paypal.me/csaeum)

---

## 🔗 Liens

- [Documentation Portainer](https://docs.portainer.io/)
- [Documentation Traefik](https://doc.traefik.io/traefik/)
- [Documentation Docker](https://docs.docker.com/)
