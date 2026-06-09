---
name: drupal-gin — setup
description: Installer et configurer le thème Gin comme admin theme Drupal - installation, activation, settings (navigation, dark mode, accent color), et modules complémentaires.
---

# Gin Setup — Installation et Configuration

## Installation

```bash
# gin_toolbar et gin_login sont des projets contrib SÉPARÉS (pas des sous-modules de gin)
composer require drupal/gin drupal/gin_toolbar drupal/gin_login
drush theme:enable gin -y
drush en gin_toolbar gin_login -y

# Définir Gin comme thème admin (clé 'admin', PAS 'default')
drush config:set system.theme admin gin -y
drush cr

# Pour la sidebar moderne (D10.3+), activer le module Navigation core et basculer Gin :
drush en navigation -y
drush config:set gin.settings settings.classic_toolbar new -y
drush cr
```

---

## Configuration UI

```
/admin/appearance/settings/gin

Sections principales :
├── Gin Toolbar         → Navigation : horizontal | vertical | new (module Navigation core)
├── Appearance
│     ├── Dark mode     → Off / On / Automatique (suit prefers-color-scheme)
│     └── Accent color  → Presets (Blue, Green, Orange...) ou custom hex
├── Logo
│     ├── Use default   → logo Gin
│     └── Custom logo   → upload SVG/PNG
├── Content form
│     └── Meta sidebar  → Metadonnées (statut, date, auteur) en sidebar droite
└── Sticky action bar   → Barre d'actions flottante sur les formulaires longs
```

---

## Exporter la Configuration Gin

```bash
# Exporter — capture les settings Gin dans la config YAML
drush cex -y

# Fichiers générés :
# config/sync/gin.settings.yml
# config/sync/system.theme.yml

# Committer pour partager entre développeurs
git add config/sync/gin.settings.yml config/sync/system.theme.yml
git commit -m "config: thème admin Gin configuré"
```

```yaml
# gin.settings.yml exemple
langcode: fr
status: true
id: gin
theme: gin
settings:
  enable_darkmode: '0'          # 0=Off, 1=On, 2=Auto (suit prefers-color-scheme)
  classic_toolbar: horizontal   # horizontal | vertical | new
  preset_accent_color: blue     # blue, teal, green, purple, pink, orange, red... ou 'custom'
  accent_color: '#0678BE'       # utilisé seulement si preset_accent_color = custom
  preset_focus_color: gin       # gin | green | claro | high_contrast | custom
  high_contrast_mode: false
  show_description_toggle: false
  sticky_action_buttons: '1'
  show_user_theme_settings: true  # L'utilisateur peut personnaliser dark mode / accent
```

---

## Gin Login — Page de Connexion Branded

```bash
drush en gin_login -y
```

```
/admin/appearance/settings/gin_login

Configuration :
  ├── Logo sur la page login
  ├── Image de fond (background)
  ├── Description de l'application
  └── Lien vers le site frontend
```

# Toute la config gin_login se fait via l'UI ci-dessus puis `drush cex`.
# Les noms de clés YAML varient selon la version : ne pas les écrire à la main,
# laisser Drupal les générer. Inspecter le résultat avec :
#   drush config:get gin_login.settings

---

## Gin Toolbar — Navigation Améliorée

```bash
drush en gin_toolbar -y
```

```
Valeurs classic_toolbar dans les settings Gin :
  ├── horizontal  (barre en haut — comme Claro)
  ├── vertical    (toolbar latérale gauche fournie par gin_toolbar)
  └── new         (délègue au module core "navigation" — sidebar moderne D10.3+)
```

---

## Permissions Gin — Personnalisation par Utilisateur

```
/admin/people/permissions → section "Gin"

Permissions :
  ├── "Allow users to use the Gin user settings form"
  │     → Les utilisateurs peuvent choisir leur dark mode / accent color
  └── "Administer Gin theme settings"
        → Accès aux settings globaux Gin
```

---

## Modules Compatibles Gin

| Module | Usage |
|--------|-------|
| `drupal/gin_toolbar` | Toolbar Gin (requis, projet contrib séparé) |
| `navigation` (core) | Sidebar moderne D10.3+ (`classic_toolbar: new`) |
| `drupal/gin_login` | Page de connexion branded |
| `drupal/field_group` | Grouper les champs en onglets dans les formulaires |
| `drupal/layout_paragraphs` | Interface Paragraphs compatible Gin |
| `drupal/media_library_form_element` | Media Library dans les formulaires custom |
| `drupal/gin_gutenberg` | Editeur Gutenberg compatible Gin |

---

## Vérification Post-Installation

```bash
# Vérifier que Gin est actif
drush php:eval "echo \Drupal::config('system.theme')->get('admin');"
# → 'gin'

# Vérifier les settings
drush config:get gin.settings

# Vérifier qu'aucun conflit CSS
# Ouvrir /admin/content — doit afficher l'interface Gin
```
