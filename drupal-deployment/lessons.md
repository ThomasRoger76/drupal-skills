# Leçons — drupal-deployment

Incidents de déploiement réels. Mis à jour après chaque résolution.

---

### 2026-05-16 — `drush cim` avant `drush updb` — DB incohérente

- **Symptôme :** Erreur PHP fatale après déploiement car la config référence des champs non créés
- **Cause :** `drush cim` importé avant `drush updb` — la config D10 était appliquée sur un schéma D9
- **Correct :** Toujours dans l'ordre : `drush updb -y && drush cim -y && drush cr` → ou simplement `drush deploy`
- **Prévention :** Utiliser `drush deploy` exclusivement — il garantit l'ordre correct

### 2026-05-16 — `composer update` en production — version inattendue installée

- **Symptôme :** Un module s'est mis à jour vers une version breaking changes en production
- **Cause :** `composer update` lancé en production au lieu de `composer install`
- **Correct :** `composer install` respecte le lock file. `composer update` l'ignore.
- **Prévention :** Règle : `composer update` JAMAIS en production. Toujours `composer install`.

### 2026-05-16 — Maintenance mode oublié actif après déploiement

- **Symptôme :** Site en maintenance mode pendant 2 heures en production
- **Cause :** Script de déploiement activait le maintenance mode mais ne le désactivait pas en cas d'erreur
- **Correct :** Utiliser `trap` bash pour désactiver en cas d'erreur : `trap 'drush state:set system.maintenance_mode 0' ERR`
- **Prévention :** Scripts de déploiement doivent toujours désactiver le maintenance mode — même en cas d'erreur

### 2026-05-16 — Symlink non atomique — downtime pendant le switch

- **Symptôme :** 503 pendant 2 secondes lors du changement de version
- **Cause :** `mv` au lieu de `ln -sfn` pour le switch de symlink — mv n'est pas atomique
- **Correct :** `ln -sfn /new/release /current` — atomique sur les systèmes POSIX
- **Prévention :** Toujours `ln -sfn` pour le switch, jamais `mv` ou `cp`

### 2026-05-16 — Platform.sh — hook deploy qui échoue silencieusement

- **Symptôme :** `drush deploy` n'est pas exécuté après le déploiement sur Platform.sh
- **Cause :** Le hook `deploy` dans `.platform.app.yaml` avait une erreur PHP mais Platform.sh ne la remontait pas clairement
- **Correct :** Ajouter `set -e` au début du hook deploy + consulter les logs d'activité Platform.sh
- **Prévention :** `set -e` dans tous les hooks. Tester avec `platform activity:get ACTIVITY_ID --log`

### 2026-05-16 — trusted_host_patterns manquant — erreur 400 en production

- **Symptôme :** Site retourne 400 Bad Request pour toutes les requêtes après le déploiement sur le nouveau domaine
- **Cause :** `trusted_host_patterns` dans `settings.php` ne contient pas le nouveau domaine
- **Correct :** Ajouter `$settings['trusted_host_patterns'] = ['^mon-nouveau-site\.com$'];` dans le settings de prod
- **Prévention :** Checklist de déploiement : mettre à jour `trusted_host_patterns` AVANT le switch DNS

### 2026-05-16 — Rollback impossible — pas de backup avant déploiement

- **Symptôme :** Déploiement cassé, pas de moyen de revenir en arrière sans perte de données
- **Cause :** Pas de dump DB avant le déploiement — `drush updb` a modifié des tables de façon irréversible
- **Correct :** Restaurer depuis le backup quotidien automatisé (si configuré)
- **Prévention :** `drush sql:dump --gzip` TOUJOURS avant `drush deploy` en production. Script atomique avec backup.

### 2026-06-08 — `gunzip dump.sql.gz | drush sql:cli` n'importe rien

- **Symptôme :** Import DB silencieusement vide après un `platform db:dump` / `ssh prod drush sql:dump`
- **Cause :** `gunzip fichier.gz` décompresse **en place** et n'écrit rien sur stdout — le pipe vers `drush sql:cli` reçoit un flux vide
- **Correct :** `gunzip -c fichier.sql.gz | drush sql:cli` (ou `zcat fichier.sql.gz | ...`)
- **Prévention :** Toujours `-c` (stdout) quand `gunzip` est en amont d'un pipe.

### 2026-06-08 — DDEV proscrit — Docker natif uniquement

- **Symptôme :** Exemples de sync DB/aliases référençant DDEV (`.ddev.site`, `ddev drush`)
- **Cause :** Standard projet : Docker natif (`docker compose exec php drush …`), jamais DDEV
- **Correct :** `docker compose exec php drush sql:sync @prod @self` · URI locale `*.localhost`
- **Prévention :** Aucune occurrence de `ddev` dans les skills Drupal — `docker compose exec php` partout.
