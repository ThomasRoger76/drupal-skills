---
name: drupal-multilingual — config translation
description: Traduire la configuration Drupal (menus, blocs, Views, text formats, roles) avec le module Configuration Translation - UI et API PHP.
---

# Configuration Translation — Référence Complète

## Qu'est-ce que la Config Translation

Le module Configuration Translation permet de traduire les chaînes de la **configuration Drupal** (pas le contenu, pas l'interface PHP).

```
Contenu traduit par Content Translation :
  → Nœuds, termes, médias, entités

Interface traduite par Interface Translation (locale) :
  → Chaînes PHP (t('...'), module strings, theme strings)

Configuration traduite par Configuration Translation :
  → Noms de menus, labels de blocs, titres de Views
  → Labels de rôles, noms de Content Types
  → Formats de texte, Image Styles
  → Toute chaîne dans un fichier YAML de config
```

---

## Configuration via l'UI

```
/admin/config/regional/config-translation

→ Lister tous les types de config traduisibles :
  ├── Content types (node.type.article)
  ├── Menus (system.menu.main)
  ├── Blocks (block.block.*)
  ├── Views (views.view.frontpage)
  ├── Roles (user.role.editor)
  ├── Image styles
  └── Text formats

→ Cliquer sur "Translate" pour modifier les traductions
```

---

## API PHP — Lire et Écrire les Traductions de Config

```php
use Drupal\language\ConfigurableLanguageManagerInterface;

// Lire un override de config par langue
$language_manager = \Drupal::languageManager();

// Obtenir le language config override pour le français
$fr_override = $language_manager->getLanguageConfigOverride('fr', 'system.menu.main');
$name_fr = $fr_override->get('label');
// → "Navigation principale" (si traduit en FR)

// Écrire un override de config
$fr_override = $language_manager->getLanguageConfigOverride('fr', 'system.menu.main');
$fr_override->set('label', 'Navigation principale');
$fr_override->save();

// Traduction d'un Content Type label
$fr_override = $language_manager->getLanguageConfigOverride('fr', 'node.type.article');
$fr_override->set('name', 'Article');
$fr_override->set('description', 'Article de blog ou actualité.');
$fr_override->save();

// Vérifier si une traduction existe
$fr_override = $language_manager->getLanguageConfigOverride('fr', 'system.menu.main');
$has_translation = !$fr_override->isNew();
```

---

## Traduire les Menus

```yaml
# config/install/system.menu.main.yml — config de base (EN)
langcode: en
status: true
id: main
label: 'Main navigation'
description: 'Site section links available in the header.'
locked: false
```

```yaml
# config/install/language/fr/system.menu.main.yml — override FR
langcode: en
status: true
id: main
label: 'Navigation principale'          ← label FR
description: 'Liens de navigation du site disponibles dans l''en-tête.'
```

**Via l'UI :** `/admin/config/regional/config-translation/system.menu.main` → Translate

---

## Traduire les Blocs

```php
// Lire le label d'un bloc dans une langue donnée
$language_manager = \Drupal::languageManager();
$fr_override = $language_manager->getLanguageConfigOverride('fr', 'block.block.mon_bloc');
$label_fr = $fr_override->get('settings.label') ?? $fr_override->get('label');

// Écrire
$fr_override->set('settings.label', 'Mon Bloc en Français');
$fr_override->save();
```

---

## Traduire les Views (titre, nom, en-tête)

```yaml
# config/install/language/fr/views.view.frontpage.yml
langcode: en
label: 'Page d''accueil'                    ← Nom de la View en FR
display:
  default:
    display_options:
      title: 'Articles récents'             ← Titre de la page
      header:
        text:
          content: '<p>Bienvenue sur notre site.</p>'
```

**Via l'UI :** `/admin/config/regional/config-translation/views.view.frontpage` → Translate

---

## Traduire les Rôles

```yaml
# config/install/language/fr/user.role.editor.yml
langcode: en
label: 'Éditeur'
```

---

## Exporter/Importer les Traductions de Config

```bash
# Exporter toutes les configs (incluant les overrides de langue)
docker compose exec php drush cex -y

# Les fichiers de traduction sont dans :
# config/sync/language/fr/*.yml
# config/sync/language/de/*.yml

# Structure :
# config/sync/
# ├── system.menu.main.yml           ← config de base
# └── language/
#     ├── fr/
#     │   └── system.menu.main.yml  ← override FR
#     └── de/
#         └── system.menu.main.yml  ← override DE

# Importer
docker compose exec php drush cim -y
```

---

## Vérifier les Traductions de Config

```bash
# Lister toutes les configs qui ont des traductions
docker compose exec php drush php:eval "
\$overrides = \Drupal::service('config.factory')->listAll();
foreach (\$overrides as \$name) {
  \$fr = \Drupal::languageManager()->getLanguageConfigOverride('fr', \$name);
  if (!\$fr->isNew()) {
    echo \$name . PHP_EOL;
  }
}
" | head -20

# Vérifier un override spécifique
docker compose exec php drush php:eval "
\$override = \Drupal::languageManager()->getLanguageConfigOverride('fr', 'system.menu.main');
var_export(\$override->get());
"

# Via drush config:get avec langue
docker compose exec php drush php:eval "
\$lang_mgr = \Drupal::languageManager();
\$original = \Drupal::config('system.menu.main')->get('label');
\$fr = \$lang_mgr->getLanguageConfigOverride('fr', 'system.menu.main')->get('label');
echo 'EN: ' . \$original . PHP_EOL;
echo 'FR: ' . (\$fr ?? '(non traduit)') . PHP_EOL;
"
```
