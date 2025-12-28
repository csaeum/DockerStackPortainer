# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- Initial release of Portainer Docker Stack
- Full Traefik v2+ integration with Let's Encrypt SSL
- Persistent volume configuration using bind-mount
- Comprehensive documentation in German, English, and French
- CI/CD pipeline with GitHub Actions
  - Docker Compose validation
  - YAML linting
  - Security scanning (gitleaks)
  - Automated stack build tests
- Automated release workflow with changelog generation
- Example configuration file (.env.example)
- Security best practices documentation
- Backup and restore instructions

### Security
- .env file excluded from git tracking
- Traefik middleware support for:
  - HTTP to HTTPS redirect
  - Geo-blocking (23 high-risk countries)
  - Security headers (HSTS, X-Frame-Options)
  - Rate-limiting (DoS protection)
  - Compression (Gzip/Brotli)

## [1.0.0] - YYYY-MM-DD

### Added
- Initial stable release

---

**Note:** Replace YYYY-MM-DD with actual release date when creating a release.
