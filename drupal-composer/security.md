---
name: drupal-composer — security
description: Gérer la sécurité des dépendances Drupal avec Composer - composer audit, drush pm:security, intégration CI/CD, et workflows de mise à jour de sécurité.
---

# Sécurité Composer — Référence Complète

## composer audit — Vulnérabilités PHP

```bash
# Scanner toutes les dépendances pour des CVE connus
composer audit

# Scanner sans les dépendances de dev
composer audit --no-dev

# Format JSON pour les outils CI
composer audit --format=json

# Exemple de sortie :
# Found 2 security vulnerability advisories affecting 2 packages.
# drupal/core  CVE-2024-XXXX  ...
# drupal/token SA-CONTRIB-2024-XXX ...
```

**Intégration dans le workflow :**
```bash
# Dans Makefile — vérifier avant déploiement
security-check:
	composer audit --no-dev
	drush pm:security

# Bloquer si vulnérabilité critique détectée
composer audit --no-dev --abandoned=fail
```

---

## drush pm:security — Advisories Drupal

```bash
# Vérifier les modules vulnérables depuis drupal.org Security Advisories
drush pm:security

# Format JSON (pour CI/CD)
drush pm:security --format=json

# Exemple de sortie :
# drupal/paragraphs 1.14.0 has a security advisory:
#   SA-CONTRIB-2024-003 — Access Bypass
#   https://www.drupal.org/sa-contrib-2024-003
```

---

## Workflow de Mise à Jour de Sécurité

```bash
# 1. Identifier les mises à jour de sécurité disponibles
drush pm:security
composer audit --no-dev

# 2. Backup DB avant mise à jour
drush sql:dump --gzip --result-file=backup-$(date +%Y%m%d).sql.gz

# 3. Mettre à jour le(s) module(s) vulnérable(s)
composer update drupal/paragraphs --no-install
composer install  # Installe depuis composer.lock mis à jour

# 4. Appliquer les updates DB si nécessaire
drush updb -y
drush cr

# 5. Tester
drush core:requirements --severity=2
curl -s -o /dev/null -w "%{http_code}" https://mon-site.com/

# 6. Déployer
git add composer.json composer.lock
git commit -m "security: update drupal/paragraphs (SA-CONTRIB-2024-003)"
git push

# 7. Déployer en production
# (selon workflow CI/CD)
```

---

## Intégration CI/CD — Bloquer si Vulnérable

```yaml
# .gitlab-ci.yml — stage de sécurité
security:composer-audit:
  stage: validate
  image: php:8.3-cli
  script:
    - curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer
    - composer audit --no-dev --format=json > security-report.json
    - |
      CRITICAL=$(jq '[.advisories[] | select(.severity == "critical")] | length' security-report.json)
      if [ "$CRITICAL" -gt "0" ]; then
        echo "ÉCHEC : $CRITICAL vulnérabilité(s) critique(s) trouvée(s)"
        cat security-report.json | jq '.advisories[] | select(.severity == "critical")'
        exit 1
      fi
  artifacts:
    when: always
    paths:
      - security-report.json
    expire_in: 1 week

security:drupal-advisories:
  stage: validate
  script:
    - vendor/bin/drush pm:security --format=json > drupal-security.json || true
    - |
      CRITICAL=$(jq 'length' drupal-security.json 2>/dev/null || echo "0")
      if [ "$CRITICAL" -gt "0" ]; then
        echo "ÉCHEC : $CRITICAL advisory Drupal détecté"
        cat drupal-security.json
        exit 1
      fi
```

---

## Monitorer les Mises à Jour de Sécurité

```bash
# Configurer les notifications email (interface Drupal)
# /admin/config/system/updates → "Notify by email"

# Vérifier l'état des mises à jour disponibles
drush pm:list --status=enabled --format=json | \
  jq '.[] | select(.security_update == true) | .name'

# Script de rapport quotidien (cron)
#!/bin/bash
# /etc/cron.daily/drupal-security-check
cd /var/www/mon-site
ISSUES=$(vendor/bin/drush pm:security --format=json 2>/dev/null | jq 'length')
if [ "$ISSUES" -gt "0" ]; then
  echo "$ISSUES advisory Drupal détecté sur mon-site" | \
    mail -s "ALERTE SÉCURITÉ Drupal" admin@mon-site.com
fi
```

---

## Packages Abandonnés

```bash
# Détecter les packages abandonnés (plus maintenus)
composer outdated --direct 2>&1 | grep "abandoned"

# Dans CI — traiter les packages abandonnés comme des erreurs
composer audit --abandoned=fail

# Alternatives courantes :
# drupal/adminimal_theme → drupal/gin (plus maintenu)
# drupal/features → drupal/configuration_management
# webflo/drupal-core-require-dev → drupal/core-dev
```

---

## auth.json — Sécurité des Credentials

```bash
# auth.json contient des tokens sensibles — NE JAMAIS COMMITTER
echo "auth.json" >> .gitignore

# Vérifier que auth.json n'est pas dans git
git check-ignore auth.json
# → doit retourner "auth.json"

# En CI/CD — utiliser des variables d'environnement
export COMPOSER_AUTH='{"github-oauth":{"github.com":"$TOKEN"}}'
```
