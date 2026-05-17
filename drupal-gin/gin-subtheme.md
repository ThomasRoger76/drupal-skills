---
name: drupal-gin — sous-thème
description: Créer un sous-thème Gin pour personnaliser l'interface d'administration Drupal - CSS custom, logo, couleurs, et mise en page des formulaires d'édition.
---

# Gin Sub-theme — Personnalisation

## Créer un Sous-Thème Gin

```
web/themes/custom/mon_admin_theme/
├── mon_admin_theme.info.yml
├── mon_admin_theme.libraries.yml
├── mon_admin_theme.theme
└── css/
    └── admin-overrides.css
```

---

## `.info.yml` — Déclarer le Sous-Thème

```yaml
# mon_admin_theme.info.yml
name: 'Mon Admin Theme'
type: theme
description: 'Personnalisation Gin pour Mon Projet'
core_version_requirement: '^10 || ^11'
base theme: gin
libraries:
  - mon_admin_theme/global-admin
```

---

## `.libraries.yml` — CSS/JS Custom

```yaml
# mon_admin_theme.libraries.yml

global-admin:
  version: 1.0
  css:
    theme:
      css/admin-overrides.css: {}
```

---

## CSS d'Override

```css
/* css/admin-overrides.css */

/* ── Variables de couleur Gin (override des variables CSS) ──────────────── */
:root {
  --gin-color-primary: #1a5276;           /* Bleu foncé custom */
  --gin-color-primary-light: #2e86c1;
  --gin-color-primary-lightest: #d6eaf8;
  --gin-color-primary-dark: #154360;

  /* Logo */
  --gin-logo-url: url('../logo.svg');
}

/* ── Toolbar / Navigation ───────────────────────────────────────────────── */
/* Ajouter le logo du client dans la sidebar */
.gin-sidebar-toggle .gin-logo {
  background-image: var(--gin-logo-url);
  background-size: contain;
  background-repeat: no-repeat;
}

/* ── Formulaires d'édition ──────────────────────────────────────────────── */
/* Personnaliser la largeur du formulaire d'édition */
.layout-container .gin--content-form {
  max-width: 1200px;
}

/* Mise en évidence des champs obligatoires */
.form-required::after {
  color: var(--gin-color-danger);
  font-weight: bold;
}

/* ── Tableau d'administration ───────────────────────────────────────────── */
/* Zebra striping sur les tables admin */
.views-table tbody tr:nth-child(even) {
  background-color: rgba(0, 0, 0, 0.02);
}
```

---

## Mise en Page des Formulaires (Meta Sidebar)

```
Gin → Settings → Content form → Meta sidebar

Avec "Meta sidebar" activé :
  ├── Colonne principale (70%) → les champs de contenu
  └── Colonne droite (30%) → statut, auteur, date de publication, révision

Pour personnaliser quels champs vont en sidebar :
  → Module "field_group" + Group type "Details (Sidebar)"
```

```php
// Déplacer des champs en sidebar via hook_form_alter
function mon_admin_theme_form_node_article_edit_form_alter(array &$form, $form_state): void {
  // Déplacer le champ SEO dans le panneau méta
  if (isset($form['field_seo_title'])) {
    $form['field_seo_title']['#group'] = 'meta';
  }
}
```

---

## Activer le Sous-Thème

```bash
# Activer le sous-thème
drush theme:enable mon_admin_theme -y

# Définir comme thème admin
drush config:set system.theme admin mon_admin_theme -y
drush cr

# Vérifier
drush php:eval "echo \Drupal::config('system.theme')->get('admin');"
# → 'mon_admin_theme'
```

---

## Préserver la Configuration entre Mises à Jour

```bash
# La config du sous-thème est dans composer.json / git
# À chaque mise à jour de Gin :
composer update drupal/gin
drush cr
# Vérifier que les overrides CSS sont toujours valides
# (tester /admin/content et /admin/structure)
```
