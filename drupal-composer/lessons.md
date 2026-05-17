# Leçons — drupal-composer

Problèmes Composer rencontrés en projets Drupal réels. Mis à jour après chaque résolution.

---

## 2026-05-16 — Création du skill

### `composer update` en production — perte de la version du lock
- **Symptôme :** En production, une version différente de celle testée en CI est installée
- **Cause :** `composer update` ignore `composer.lock` et installe les dernières versions compatibles
- **Correct :** En production TOUJOURS `composer install` — respecte le lock file
- **Prévention :** `composer install --no-interaction` en CI et en déploiement. `composer update` uniquement en développement local

### Patch qui ne s'applique plus après mise à jour du module
- **Symptôme :** `composer install` échoue avec "Could not apply patch"
- **Cause :** La mise à jour du module a modifié les lignes patchées
- **Correct :** 1) Vérifier si le patch est inclus dans la nouvelle version (consulter le CHANGELOG). 2) Si non → chercher une version mise à jour du patch sur drupal.org. 3) Si inexistant → créer un nouveau patch depuis la version actuelle
- **Prévention :** Après chaque `composer update`, toujours vérifier que les patches s'appliquent

### `composer.lock` absent du git — versions différentes entre devs
- **Symptôme :** "Ça marche sur ma machine" — des modules installés en versions différentes selon les développeurs
- **Cause :** `composer.lock` dans `.gitignore` ou non commité
- **Correct :** Committer `composer.lock`. `composer install` utilise le lock pour tous
- **Prévention :** Vérifier `.gitignore` ne contient pas `composer.lock`. Ajouter au pre-commit hook

### Module custom non trouvé — `type: drupal-module` manquant dans composer.json du module
- **Symptôme :** `composer require` échoue avec "Package not found"
- **Cause :** Le `composer.json` du module custom n'a pas `"type": "drupal-module"` — Composer ne sait pas où l'installer
- **Correct :** Ajouter `"type": "drupal-module"` dans le `composer.json` du module custom
- **Prévention :** Template de module custom avec `composer.json` incluant le type correct

### `allow-plugins` manquant — Composer 2.2+ refuse d'exécuter les plugins
- **Symptôme :** `composer install` échoue avec "The following plugins were not loaded due to missing allow-plugins"
- **Cause :** Composer 2.2+ exige une liste explicite des plugins autorisés dans `config.allow-plugins`
- **Correct :** Ajouter dans `composer.json` → `"config": {"allow-plugins": {"cweagans/composer-patches": true, ...}}`
- **Prévention :** Toujours inclure `allow-plugins` pour tous les plugins utilisés dans le template de projet

### `composer install` très lent en Docker — pas de cache monté
- **Symptôme :** `composer install` prend 5-15 minutes en Docker même sans changements
- **Cause :** Le cache Composer est dans le container — effacé à chaque rebuild
- **Correct :** Monter le cache Composer : `~/.composer:/home/www-data/.composer` dans docker-compose.yml
- **Prévention :** Template docker-compose.yml avec le volume Composer monté par défaut

### COMPOSER_AUTH en clair dans docker-compose.yml — fuite de credentials
- **Symptôme :** Token GitLab visible dans `docker-compose.yml` commité
- **Cause :** `COMPOSER_AUTH` défini directement dans la valeur du service Docker
- **Correct :** `COMPOSER_AUTH: ${COMPOSER_AUTH}` dans docker-compose.yml → valeur dans `.env` (gitignored)
- **Prévention :** Règle hookify sur les patterns de tokens dans docker-compose.yml
