---
name: drupal-gin — setup
description: Installer et configurer le thème Gin comme admin theme Drupal - installation, activation, settings (navigation, dark mode, accent color), et modules complémentaires.
---

# Gin Setup — Installation et Configuration

## Installation

```bash
composer require drupal/gin drupal/gin_toolbar drupal/gin_login
drush en gin gin_toolbar gin_login -y

# Définir Gin comme thème admin
drush config:set system.theme admin gin -y
drush cr
```

---

## Configuration UI

```
/admin/appearance/settings/gin

Sections principales :
├── Gin Toolbar         → Position : horizontal (classique) | sidebar | condensed
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
  enable_darkmode: '0'          # 0=Off, 1=On, 2=Auto
  classic_toolbar: horizontal   # horizontal|sidebar|condensed
  accent_color: claro_blue      # Preset ou custom_color
  preset_accent_color: gin_accent_blue
  custom_accent_color: '#0678BE'
  high_contrast_mode: false
  show_description_toggle: false
  sticky_action_buttons: false
  secondary_toolbar_enabled: false
  content_form_width: full     # full|md|sm
  sidebar_expand_all: false
  show_user_theme_settings: true  # L'utilisateur peut personaliser
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

```yaml
# gin_login.settings.yml
background_image: ''
logo_path: 'themes/custom/mon_sous_theme_gin/logo.svg'
description: 'Espace d''administration — Mon Projet'
```

---

## Gin Toolbar — Navigation Améliorée

```bash
drush en gin_toolbar -y
```

```
Options de toolbar dans les settings Gin :
  ├── Horizontal  (barre en haut — comme Seven/Claro)
  ├── Sidebar     (menu latéral gauche — style backend moderne)
  └── Condensed   (sidebar avec icônes seulement)
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
| `drupal/gin_toolbar` | Navigation sidebar améliorée |
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
