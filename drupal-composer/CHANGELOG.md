# Changelog — drupal-composer

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
