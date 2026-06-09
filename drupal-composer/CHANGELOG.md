# Changelog — drupal-composer

---

## v1.1 — 2026-06-09

**Audit qualité — corrections factuelles et currency**

### Corrigé
- **`patches.md`** — section `composer-patches` v2 réécrite : la v2 est stable (`^2.0`, plus de `@beta`) ; le format étendu correct est une **liste d'objets** sous `extra.patches` avec champ `url` (l'ancien texte utilisait à tort `extra.composer-patches` + `source` + structure objet). Ajout des pièges de migration v1→v2 et du `patches.lock.json`.
- **`patches.md`** — commande de listing des patches corrigée (`composer config extra.patches` / `jq` au lieu d'un `composer show -d .` qui ne sort pas le JSON).
- **`version-constraints.md`** — commentaire de `composer update --lock` clarifié (resync du content-hash, ne met pas à jour les versions).

### Ajouté
- **`SKILL.md`** — note Docker natif (`docker compose exec php composer …`, jamais DDEV) dans la règle fondamentale.
- **`SKILL.md`** — entrées Quick Decision Table : `composer bump`, patches v2.
- **`version-constraints.md`** — section `composer bump` (Composer 2.4+).
- **`composer-basics.md`** — note de currency D11 (passage `^10` → `^11`, Drush `^13`, PHP 8.3+).
- **`lessons.md`** — 2 leçons (format v2, `update --lock` vs `bump`).

---

## v1.0 — 2026-05-16

**Création initiale — skill critique manquant identifié lors de l'audit**

### Couverture

**`SKILL.md`**
- Quick Decision Table (25+ entrées couvrant install, patches, repos privés, versions, déploiement)
- Anti-patterns critiques (8 entrées)
- Tableau versioning Composer 1 vs 2 × Drupal

**`composer-basics.md`**
- Structure complète du `composer.json` Drupal standard (annoté)
- `drupal/core-recommended` vs `drupal/core`
- Scaffold — protéger les fichiers custom
- Toutes les commandes essentielles (require, update, remove, show, why, why-not)
- Scripts Composer pour l'automatisation du déploiement
- Optimisation autoloader production

**`patches.md`**
- `cweagans/composer-patches` — installation et configuration
- Patches drupal.org avec convention de nommage (#ISSUE_ID)
- Patches locaux — création depuis diff
- `patches-file` pour projets avec beaucoup de patches
- `composer-patches` v2 avec sha256 verification
- Forks comme alternative aux patches lourds
- Diagnostic des patches qui échouent (`--prefer-source`, fuzz)
- Gestion post-mise à jour des patches

**`version-constraints.md`**
- Syntaxe complète (`^`, `~`, `*`, ranges, stabilité)
- Contraintes recommandées par cas d'usage
- Commandes d'upgrade core D9→D10→D11 exactes
- `composer why-not` pour diagnostiquer les conflits
- Résolution de conflits (4 stratégies)
- `composer.lock` — bonnes pratiques

**`private-repos.md`**
- Repository Git privé (GitLab/GitHub) avec `auth.json`
- `COMPOSER_AUTH` en variable d'environnement (CI/CD)
- Satis — repository Composer privé complet
- Module custom en monorepo (path repository)
- CI/CD — variables pour GitLab CI et GitHub Actions

**`deployment.md`**
- Stratégie de déploiement (dev → CI → prod)
- Commande production complète avec toutes les options
- Cache Composer en CI/CD (GitLab + GitHub Actions)
- Cache dans Docker (volume monté)
- Scripts Composer pour workflow
- Déploiement zéro-temps d'arrêt (atomic avec symlinks)
- Vérifications post-déploiement

**`lessons.md`**
- 7 incidents Composer réels avec corrections

---

## Compatibilité

| Skill version | Composer | Drupal | Notes |
|--------------|---------|--------|-------|
| v1.0 | 2.x | D8, D9, D10, D11 | Composer 2 requis D9+, allow-plugins D2.2+ |
| v1.1 | 2.4+ | D8, D9, D10, D11 | composer-patches v2 stable, `composer bump`, Docker natif |
