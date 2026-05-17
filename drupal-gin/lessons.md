# Leçons — drupal-gin

Problèmes avec le thème Gin découverts en projet Drupal.

---

### 2026-05-16 — Modifications CSS perdues après mise à jour Gin

- **Symptôme :** Les CSS custom sur l'admin ont disparu après `composer update drupal/gin`
- **Cause :** CSS modifié directement dans les fichiers du module Gin (dans vendor/)
- **Correct :** Créer un sous-thème Gin avec son propre CSS → jamais modifier vendor/
- **Prévention :** Toujours créer un sous-thème avant toute personnalisation CSS

### 2026-05-16 — Gin non exporté — interface différente entre développeurs

- **Symptôme :** Un développeur voit la sidebar navigation, un autre voit la toolbar horizontale
- **Cause :** `gin.settings.yml` non commité dans git (ou settings Gin modifiés sans `drush cex`)
- **Correct :** `drush cex -y` après configuration de Gin → committer `gin.settings.yml`
- **Prévention :** Gin fait partie de la config → toujours exporter et committer

### 2026-05-16 — Conflit CSS entre Gin et un module contrib

- **Symptôme :** Les boutons d'action dans un module tiers ont un style cassé sous Gin
- **Cause :** Le module contrib a des styles CSS spécifiques à Seven/Claro non compatibles Gin
- **Correct :** Ajouter un CSS override dans le sous-thème Gin pour ce module spécifique
- **Prévention :** Tester tous les modules contrib visuellement après installation de Gin

### 2026-05-16 — Gin Login non activé sur l'environnement de staging

- **Symptôme :** La page de login est Claro en staging mais Gin Login en prod → expérience incohérente
- **Cause :** Module `gin_login` non inclus dans la config sync ou non activé sur tous les envs
- **Correct :** `drush en gin_login -y` + `drush cex` + committer dans tous les environnements
- **Prévention :** Gin Login fait partie de la config → exporter après activation

### 2026-05-16 — Gin + Media Library — boutons déplacés

- **Symptôme :** La Media Library ne s'ouvre pas correctement sous Gin
- **Cause :** Conflit CSS entre certaines versions de Gin et Media Library
- **Correct :** Mettre à jour `drupal/gin` vers la dernière version — les fixes sont fréquents
- **Prévention :** Après chaque `composer update drupal/gin`, tester l'ouverture de la Media Library

### 2026-05-16 — Variables CSS toolbar Gin non overridées dans le sous-thème

- **Symptôme :** Les couleurs custom s'appliquent sur les formulaires mais pas sur la toolbar Gin
- **Cause :** La toolbar utilise `--gin-toolbar-*` variables distinctes de `--gin-color-*`
- **Correct :** Surcharger `--gin-toolbar-bg-color`, `--gin-toolbar-text-color` dans le sous-thème
- **Prévention :** Inspecter DevTools sur la toolbar pour identifier les variables CSS spécifiques à surcharger

### 2026-05-16 — gin.settings non versionné — mise en page différente entre devs

- **Symptôme :** Toolbar horizontale chez un dev, sidebar chez un autre
- **Cause :** `gin.settings.yml` non commité — chaque dev a sa config locale
- **Correct :** `drush cex -y && git add config/sync/gin.settings.yml && git commit`
- **Prévention :** `gin.settings.yml` est de la config Drupal → toujours versionner
