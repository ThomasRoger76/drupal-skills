# Le Pipeline Preprocess

## Règle Fondamentale

**Logique dans le `.theme`, markup dans les templates Twig.**

- `.theme` = préparer les variables, calculer des classes, charger des données
- `.twig` = afficher les variables préparées, structure HTML uniquement

`\Drupal::` dans le `.theme` est **acceptable** car les fonctions preprocess sont procédurales (pas des classes). C'est l'un des rares endroits où `\Drupal::` est toléré.

---

## Nommage et Signature

```php
// Dans mon_theme.theme
// Convention : NOMTHEME_preprocess_HOOK(&$variables)
// HOOK = le nom du hook theme (node, block, page, field, html...)

function mon_theme_preprocess_node(array &$variables): void { }
function mon_theme_preprocess_page(array &$variables): void { }
function mon_theme_preprocess_block(array &$variables): void { }
function mon_theme_preprocess_field(array &$variables): void { }
function mon_theme_preprocess_html(array &$variables): void { }
function mon_theme_preprocess_views_view(array &$variables): void { }
function mon_theme_preprocess_paragraph(array &$variables): void { }  // module paragraphs
```

---

## `hook_preprocess_page` — Variables Disponibles

```php
function mon_theme_preprocess_page(array &$variables): void {
  // Variables standards disponibles :
  // $variables['page']          → régions de la page (render arrays)
  // $variables['is_front']      → bool : page d'accueil
  // $variables['logged_in']     → bool : utilisateur connecté
  // $variables['root_path']     → chemin de la route actuelle
  // $variables['node']          → nœud courant (si page de nœud)
  // $variables['db_is_active']  → bool : connexion DB active
  // $variables['user']          → utilisateur courant (entité)

  // Ajouter le slogan du site
  $variables['site_slogan'] = \Drupal::config('system.site')->get('slogan');

  // Ajouter la langue courante
  $variables['current_language'] = \Drupal::languageManager()
    ->getCurrentLanguage()
    ->getId();

  // Classe CSS selon le type de page
  if ($variables['is_front']) {
    $variables['attributes']['class'][] = 'page--front';
  }
  if (isset($variables['node'])) {
    $node = $variables['node'];
    $variables['attributes']['class'][] = 'page--type-' . $node->bundle();
  }

  // Passer des données vers JS
  $variables['#attached']['drupalSettings']['monTheme']['isLogged'] = $variables['logged_in'];
}
```

---

## `hook_preprocess_node` — Variables Disponibles

```php
function mon_theme_preprocess_node(array &$variables): void {
  /** @var \Drupal\node\NodeInterface $node */
  $node      = $variables['node'];
  $view_mode = $variables['view_mode'];  // 'full', 'teaser', 'card', etc.

  // Variables standards :
  // $variables['content']        → render array des champs
  // $variables['label']          → titre du nœud
  // $variables['url']            → URL canonique
  // $variables['page']           → bool : affiché comme page entière
  // $variables['display_submitted'] → bool : afficher auteur/date
  // $variables['author_name']    → nom de l'auteur (formé)
  // $variables['author_picture'] → render array photo auteur
  // $variables['date']           → date formatée (par défaut)
  // $variables['attributes']     → attributs HTML de l'article

  // Formatter la date en français
  $variables['date_formatted'] = \Drupal::service('date.formatter')->format(
    $node->getCreatedTime(),
    'custom',
    'd/m/Y'
  );

  // Variable de classe dynamique
  $variables['is_author'] = $node->getOwnerId() === \Drupal::currentUser()->id();
  if ($variables['is_author']) {
    $variables['attributes']['class'][] = 'node--own';
  }

  // Données du champ référence (entité liée)
  if (!$node->field_categorie->isEmpty()) {
    $term = $node->field_categorie->entity;
    $variables['categorie_label'] = $term ? $term->label() : NULL;
  }

  // Charger une librairie conditionnellement selon le type
  if ($node->bundle() === 'article' && $view_mode === 'full') {
    $variables['#attached']['library'][] = 'mon_theme/article-full';
  }
}
```

---

## `hook_preprocess_block` — Variables Disponibles

```php
function mon_theme_preprocess_block(array &$variables): void {
  // Variables standards :
  // $variables['elements']          → render array complet
  // $variables['plugin_id']         → ex: 'system_branding_block'
  // $variables['base_plugin_id']    → plugin de base (sans dérivé)
  // $variables['derivative_plugin_id'] → partie dérivée
  // $variables['configuration']     → config du bloc (label, region, etc.)
  // $variables['label']             → label du bloc (si affiché)
  // $variables['content']           → contenu du bloc
  // $variables['attributes']        → attributs HTML du wrapper

  // Ajouter une classe selon l'ID du plugin
  $variables['attributes']['class'][] = 'block--' . str_replace('_', '-', $variables['base_plugin_id']);

  // Ajouter une classe selon la région
  if (isset($variables['configuration']['region'])) {
    $variables['attributes']['class'][] = 'block--region-' . $variables['configuration']['region'];
  }

  // Logique pour un bloc spécifique
  if ($variables['plugin_id'] === 'system_branding_block') {
    $variables['site_name'] = \Drupal::config('system.site')->get('name');
  }
}
```

---

## `hook_preprocess_html` — Head & Body Classes

```php
function mon_theme_preprocess_html(array &$variables): void {
  // Variables standards :
  // $variables['html_attributes']   → attributs de <html>
  // $variables['body_attributes']   → attributs de <body>
  // $variables['head_title']        → titre de la page
  // $variables['page_top']          → contenu avant <body>
  // $variables['page_bottom']       → contenu après </body>

  // Ajouter la langue à <html>
  $variables['html_attributes']['lang'] = \Drupal::languageManager()
    ->getCurrentLanguage()
    ->getId();

  // Classes dynamiques sur <body>
  $route = \Drupal::routeMatch()->getRouteName();
  $variables['body_attributes']['class'][] = 'route--' . str_replace(['.', '_'], '-', $route);

  // Classe selon le rôle utilisateur
  $current_user = \Drupal::currentUser();
  if ($current_user->hasRole('editor')) {
    $variables['body_attributes']['class'][] = 'role--editor';
  }
}
```

---

## Multilingue dans le Preprocess — `entity.repository`

Sur les sites multilingues, le nœud reçu dans le preprocess est souvent dans sa langue **d'origine**, pas dans la langue de l'interface courante. Utiliser `entity.repository` :

```php
// ✅ Récupérer la traduction dans la langue de l'interface
function mon_theme_preprocess_page(array &$variables): void {
  $node = \Drupal::routeMatch()->getParameter('node');
  if (!$node instanceof \Drupal\node\NodeInterface) {
    return;
  }

  // getTranslationFromContext = langue de l'interface courante (pas la langue d'origine)
  $node = \Drupal::service('entity.repository')->getTranslationFromContext($node);

  // Ajouter le cache context langue
  $variables['#cache']['contexts'][] = 'languages:language_interface';
  $variables['titre_traduit'] = $node->label();
}
```

| | `$variables['node']` | `getTranslationFromContext($node)` |
|--|---------------------|----------------------------------|
| Langue | Langue d'origine du nœud | Langue de l'interface courante |
| Multilingue | ❌ Non adapté | ✅ Correct |
| Cache context | `user` | `languages:language_interface` |

---

## `hook_preprocess_field` — Manipulation des Champs

```php
function mon_theme_preprocess_field(array &$variables): void {
  $element    = $variables['element'];
  $field_name = $variables['field_name'];    // 'field_image', 'body', etc.
  $entity     = $element['#object'];          // Entité parente (nœud, bloc, etc.)
  $entity_type = $entity->getEntityTypeId(); // 'node', 'block_content', etc.

  // Variables standards :
  // $variables['items']        → tableau des items du champ
  // $variables['multiple']     → bool : champ multi-valeur
  // $variables['label']        → label du champ
  // $variables['label_display'] → 'above', 'inline', 'hidden', 'visually_hidden'

  // Masquer le label globalement sur ce champ
  if ($field_name === 'field_image') {
    $variables['label_display'] = 'hidden';
  }

  // Ajouter une classe au wrapper selon le nom du champ
  $variables['attributes']['class'][] = 'field--' . str_replace('_', '-', $field_name);
}
```

---

## `hook_page_attachments_alter` — Modifier les Assets depuis le Thème

```php
function mon_theme_page_attachments_alter(array &$attachments): void {
  // Ajouter une meta tag
  $attachments['#attached']['html_head'][] = [
    [
      '#tag'        => 'meta',
      '#attributes' => ['name' => 'theme-color', 'content' => '#1a1a2e'],
    ],
    'meta_theme_color',
  ];

  // Ajouter un preconnect pour Google Fonts
  $attachments['#attached']['html_head_link'][][] = [
    'rel'         => 'preconnect',
    'href'        => 'https://fonts.googleapis.com',
    'crossorigin' => '',
  ];
}
```

---

## `hook_library_info_alter` — Modifier des Librairies d'Autres Modules

```php
function mon_theme_library_info_alter(array &$libraries, string $extension): void {
  // Remplacer un fichier CSS d'un module contrib par le nôtre
  if ($extension === 'views' && isset($libraries['views.module'])) {
    unset($libraries['views.module']['css']['base']['css/views.module.css']);
    $libraries['views.module']['css']['base']['css/views-override.css'] = [];
  }
}
```

---

## `hook_css_alter` — Retirer des CSS au Runtime

```php
function mon_theme_css_alter(array &$css, \Drupal\Core\Asset\AttachedAssetsInterface $assets): void {
  // Retirer un fichier CSS spécifique de la page actuelle
  $file = 'core/assets/vendor/normalize-css/normalize.css';
  if (isset($css[$file])) {
    unset($css[$file]);
  }
}
```

---

## Différence `build()` vs `preprocess`

| | `build()` dans un Controller/Plugin | `preprocess_HOOK` dans `.theme` |
|--|-----------------------------------|---------------------------------|
| **Quand** | Construction du render array | Avant que Twig ne reçoive les variables |
| **Rôle** | Structure les données, définit le theme hook | Enrichit/modifie les variables Twig |
| **Accès** | Données brutes, API Drupal | Variables Twig prêtes au rendu |
| **Exemple** | Charger des entités, construire la structure | Ajouter des classes CSS, formatter des dates |
| **DI** | Via constructeur | `\Drupal::service()` acceptable |

---

## ⚠️ Piège Critique — Preprocess sans Cache Tags = Production Cassée

C'est l'erreur numéro 1 en production Drupal. Un preprocess qui charge des données dynamiques sans déclarer les cache tags appropriés servira du contenu périmé à tous les utilisateurs.

```php
// ❌ DÉSASTRE EN PRODUCTION — données chargées une fois, servies périmées indéfiniment
function mon_theme_preprocess_node(array &$variables): void {
  $node = $variables['node'];
  // Requête DB — résultat mis en cache mais JAMAIS invalidé
  $ids = \Drupal::entityQuery('node')
    ->condition('field_category', $node->field_category->target_id)
    ->accessCheck(TRUE)
    ->range(0, 3)
    ->execute();
  $variables['related'] = Node::loadMultiple($ids);
  // ZÉRO cache tag → si un nœud related est modifié, la page reste périmée
}

// ✅ CORRECT — cache invalidé automatiquement quand n'importe quel nœud related change
function mon_theme_preprocess_node(array &$variables): void {
  $node = $variables['node'];
  $ids = \Drupal::entityQuery('node')
    ->condition('field_category', $node->field_category->target_id)
    ->accessCheck(TRUE)
    ->range(0, 3)
    ->execute();
  $related = Node::loadMultiple($ids);
  $variables['related'] = $related;

  // Propager les cache tags OBLIGATOIREMENT
  $variables['#cache']['tags'] = array_merge(
    $variables['#cache']['tags'] ?? [],
    ['node_list'],                                          // Invalider si n'importe quel node change
    array_map(fn($n) => 'node:' . $n->id(), $related),    // Invalider si ce node précis change
    $node->getCacheTags(),                                  // Tags du node courant
  );
  // Cache context : si le résultat dépend de l'utilisateur connecté
  $variables['#cache']['contexts'][] = 'user.permissions';
}
```

**Règle d'or :** pour chaque entité chargée dans un preprocess, propager ses cache tags via `$variables['#cache']['tags']`. Drupal les remonte automatiquement dans le render array final.

**Voir aussi :** `drupal-core/services-internal-api.md` — section Cache API complète (tags, contexts, max-age).
**Règle :** si tu peux faire la chose en preprocess sans charger de données lourdes → preprocess. Si tu construis une structure de données complexe → `build()` dans le plugin/controller.
