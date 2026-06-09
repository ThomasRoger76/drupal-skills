# Changelog — drupal-deployment

---

## v1.2 — 2026-06-08 (amélioration qualité 9.5, alignement docker compose / D11)

**Corrections :**
- DDEV éliminé : `docker compose exec php drush` partout (zero-downtime.md, acquia-pantheon.md)
- Bug réel `gunzip dump.gz | drush sql:cli` (flux vide) → `gunzip -c` (zero-downtime.md, platform-sh.md)
- Script atomique : backup DB préalable + `trap ... ERR` pour maintenance mode (cohérence lessons.md)
- D11 currency : `drush deploy:hook` / `hook_deploy_NAME` nommé, PHP 8.3, MySQL 8.0/MariaDB 10.6
- Table d'évolution enrichie (PHP min, DB min, hooks nommés D10.3+)
- Anti-patterns ajoutés : absence de `trap`, absence de dump DB
- Acquia pipelines : MySQL 5.7 → 8.0 (obsolescence D11)
- 2 leçons ajoutées (gunzip -c, DDEV proscrit)

---

## v1.1 — 2026-05-16 (audit complet)

**Corrections :**
- See Also mis à jour (drupal-tooling remplacé par drupal-deployment)
- Leçons enrichies (7 leçons au total)
- Fichiers manquants créés (liens QDT résolus)

---

## v1.0 — 2026-05-16

**Création initiale**

- SKILL.md avec Quick Decision Table (4 fichiers de référence)
- lessons.md avec incidents réels
