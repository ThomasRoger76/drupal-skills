# Theme Suggestions & Debugging

## Activer Twig Debug

```yaml
# web/sites/default/services.yml (copier depuis default.services.yml)
parameters:
  twig.config:
    debug: true
    auto_reload: true
    cache: false
```

```bash
docker compose exec php drush cr   # Obligatoire après modification
```

---

## Lire les Commentaires Debug HTML

Avec Twig debug activé, inspecter le source HTML du navigateur :

```html
<!-- THEME DEBUG -->
<!-- THEME HOOK: 'node' -->
<!-- FILE NAME SUGGESTIONS:
   * node--123--full.html.twig
   * node--123.html.twig
   * node--article--full.html.twig   ← créer ce fichier pour cibler article en full
   * node--article.html.twig
   * node--full.html.twig
   x node.html.twig                  ← 'x' = template actuellement utilisé
-->
<!-- BEGIN OUTPUT from 'core/themes/olivero/templates/content/node.html.twig' -->
```

**Créer le fichier le plus spécifique** qui correspond au contexte souhaité.  
**Copier** le template parent comme base, puis modifier.

> **⚠️ Underscores vs tirets :** Le commentaire debug affiche les suggestions avec `__` (underscores).  
> Les fichiers sur disque utilisent `--` (tirets). Toujours remplacer `__` par `--` quand on nomme le fichier.  
> Exemple : `node__article__full` dans le debug → `node--article--full.html.twig` sur disque.

---

## Conventions de Nommage — Référence Complète

### Pages (`page--*.html.twig`)

```
page.html.twig                        # Toutes les pages
page--front.html.twig                 # Page d'accueil (is_front)
page--node.html.twig                  # Toutes les pages de nœud
page--node--article.html.twig         # Nœuds de type article
page--node--123.html.twig             # Le nœud spécifique #123
page--admin.html.twig                 # Pages d'admin (_admin_route: TRUE)
page--user--login.html.twig           # Page de login
page--search--results.html.twig       # Résultats de recherche
page--taxonomy--term.html.twig        # Pages de termes de taxonomie
```

### Nœuds (`node--*.html.twig`)

```
node.html.twig
node--article.html.twig               # Type de contenu article
node--article--full.html.twig         # Article en mode full
node--article--teaser.html.twig       # Article en mode teaser
node--123.html.twig                   # Le nœud #123
node--123--full.html.twig             # Le nœud #123 en full
```

### Blocs (`block--*.html.twig`)

```
block.html.twig
block--system-branding-block.html.twig        # Le bloc branding
block--views-block--blog-block-1.html.twig    # Un bloc Views
block--mon-module.html.twig                   # Blocs d'un module
block--REGION.html.twig                       # Blocs d'une région ⚠️ non automatique — requiert hook_theme_suggestions_block_alter()
```

### Champs (`field--*.html.twig`)

```
field.html.twig
field--field-image.html.twig                  # Champ field_image
field--node--field-image.html.twig            # field_image sur les nœuds
field--node--field-image--article.html.twig   # field_image sur article
field--image.html.twig                        # Tous les champs de type image
```

### Views

```
views-view.html.twig                          # Container de toute View
views-view--VIEW-ID.html.twig                 # La View "blog"
views-view--VIEW-ID--DISPLAY-ID.html.twig     # La View "blog", display "page_1"
views-view-unformatted.html.twig              # Format "unformatted list"
views-view-unformatted--VIEW-ID.html.twig
views-view-list.html.twig                          # Format "HTML list"
views-view-fields.html.twig                        # Format "fields"
views-view-fields--VIEW-ID.html.twig
views-view-fields--VIEW-ID--DISPLAY-ID.html.twig   # Vue + display spécifique
```

### Autres

```
# Termes de taxonomie
taxonomy-term.html.twig
taxonomy-term--VOCABULARY.html.twig
taxonomy-term--tags.html.twig

# Utilisateurs
user.html.twig

# Paragraphes (module paragraphs) — machine name en minuscules
paragraph.html.twig
paragraph--text.html.twig            # Paragraphe de type "text" (machine name)
paragraph--image-text.html.twig      # Machine name avec tirets

# Menus
menu.html.twig
menu--main.html.twig                  # Menu principal
menu--footer.html.twig

# Commentaires
comment.html.twig
comment--node--article.html.twig

# Formulaires
form.html.twig
form--search-block-form.html.twig

# HTML (document complet)
html.html.twig
```

---

## `hook_theme_suggestions_HOOK_alter()` — Suggestions Personnalisées

Dans `mon_theme.theme` :

```php
use Drupal\node\NodeInterface;

/**
 * Suggestions de page selon le bundle du nœud courant.
 */
function mon_theme_theme_suggestions_page_alter(array &$suggestions, array $variables): void {
  // Ajouter une suggestion basée sur le type de contenu
  if ($node = \Drupal::routeMatch()->getParameter('node')) {
    if ($node instanceof NodeInterface) {
      $suggestions[] = 'page__node__' . $node->bundle();
      $suggestions[] = 'page__node__' . $node->id();
    }
  }

  // Suggestion pour les pages de terme de taxonomie
  if ($term = \Drupal::routeMatch()->getParameter('taxonomy_term')) {
    $suggestions[] = 'page__taxonomy__' . $term->bundle();
  }

  // Suggestion selon un paramètre query (ex: ?mode=embed)
  if (\Drupal::request()->query->get('mode') === 'embed') {
    $suggestions[] = 'page__embed';
  }
}

/**
 * Suggestions de nœud selon le view mode ET le bundle.
 */
function mon_theme_theme_suggestions_node_alter(array &$suggestions, array $variables): void {
  $node      = $variables['elements']['#node'];
  $view_mode = $variables['elements']['#view_mode'];

  // Plus spécifique que les suggestions automatiques
  $suggestions[] = 'node__' . $node->bundle() . '__' . $view_mode;

  // Suggestion si le nœud a un certain champ
  if ($node->hasField('field_featured') && $node->field_featured->value) {
    $suggestions[] = 'node__featured';
    $suggestions[] = 'node__' . $node->bundle() . '__featured';
  }
}

/**
 * Suggestions de bloc selon la région où il est placé.
 */
function mon_theme_theme_suggestions_block_alter(array &$suggestions, array $variables): void {
  // Suggestion basée sur la région
  if (isset($variables['elements']['#configuration']['region'])) {
    $region = $variables['elements']['#configuration']['region'];
    $suggestions[] = 'block__' . $region;
  }
}

/**
 * Catch-all — altérer les suggestions de TOUS les hooks theme.
 * Utiliser avec précaution — préférer les hooks spécifiques.
 */
function mon_theme_theme_suggestions_alter(array &$suggestions, array $variables, string $hook): void {
  // $hook = 'node', 'block', 'field', etc.
}
```

---

## Priorité des Suggestions

Drupal utilise **la suggestion la plus spécifique** disponible sur le système de fichiers.  
Ordre du plus spécifique (prioritaire) au plus générique (fallback) :

```
node--123--full.html.twig      ← le plus spécifique, utilisé en premier si présent
node--123.html.twig
node--article--full.html.twig
node--article.html.twig
node--full.html.twig
node.html.twig                 ← fallback ultime
```

Twig debug indique avec `x` le template effectivement chargé.

---

## Emplacement des Templates

Drupal cherche les templates dans cet ordre :
1. **Thème actif** (`themes/custom/mon_theme/templates/`)
2. **Base themes** (chaîne d'héritage)
3. **Module fournisseur** (ex: `node/templates/`)
4. **Core**

**Organisation recommandée dans `templates/` :**

```
templates/
  layout/
    page.html.twig
    page--article.html.twig
    html.html.twig
  content/
    node.html.twig
    node--article.html.twig
    node--article--teaser.html.twig
  block/
    block.html.twig
  field/
    field--field-image.html.twig
  navigation/
    menu.html.twig
    menu--main.html.twig
  views/
    views-view--blog.html.twig
  misc/
    status-messages.html.twig
```
