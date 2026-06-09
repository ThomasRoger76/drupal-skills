---
name: drupal-multilingual — views multilingues
description: Configurer des Views Drupal pour les sites multilingues - Translation Language, filtres par langcode, Views pour le language switcher, et debugging des résultats multilingues.
---

# Views Multilingues — Configuration Complète

## Le Problème Central

Sans configuration multilingue correcte, une View affiche **tous les contenus quelle que soit la langue** — ou pire, affiche le contenu de la mauvaise langue.

---

## Configuration Views pour Sites Multilingues

```
Views → Edit View → Advanced → Other

"Content language and Translation Language"
  → Doit être configuré pour afficher le contenu dans la bonne langue
```

### Option "Translation Language" dans les Champs

```
Field → Champ textuel → Choisir la langue
  ├── Current user's language
  ├── Content language selected for page  ← recommandé
  ├── Default site language
  └── Fixed language : French, English...
```

---

## Configuration Recommandée pour une Vue Multilingue

```yaml
# config/sync/views.view.articles_multilingue.yml
display:
  default:
    display_options:
      # Langue du contenu — utiliser la langue de la page courante
      rendering_language: '***LANGUAGE_language_content***'

      filters:
        # 1. Filtrer uniquement les nœuds publiés
        status:
          id: status
          value: '1'

        # 2. Filtrer par langue — afficher seulement les traductions disponibles
        # OPTION A : filtrer par langue courante (strict)
        langcode:
          id: langcode
          table: node_field_data
          field: langcode
          value:
            '***LANGUAGE_language_content***': '***LANGUAGE_language_content***'

        # OPTION B : afficher le nœud si traduit OU si original
        # → Configurer "Translation Language" dans les fields à la place du filtre
```

### Via l'UI — Étapes

```
1. Views → Filtres → Ajouter → "Content: Translation language"
   → Mettre "Current user's language" comme valeur par défaut

2. Pour chaque champ textuel (titre, body...) :
   Views → Champ → Language Settings
   → "Content language of the view results"

3. Views → Advanced → Other → Rendering Language
   → "Current user's language"
```

---

## Filtre par Langue Exposé

Permettre à l'utilisateur de choisir la langue des résultats :

```yaml
# Filtre langcode exposé dans une Vue admin
filters:
  langcode:
    id: langcode
    table: node_field_data
    field: langcode
    exposed: true
    expose:
      label: 'Langue'
      identifier: langcode
      multiple: false
    # Pas de valeur par défaut = afficher toutes les langues
```

---

## Views pour le Language Switcher (Détection des Traductions)

La block natif Drupal "Language switcher" détecte automatiquement les traductions disponibles. Voici comment l'implémenter manuellement :

```php
// Dans un preprocess — générer les liens de langues pour la page courante
function mon_theme_preprocess_node(array &$variables): void {
  $node = $variables['node'];
  $language_manager = \Drupal::languageManager();
  $languages = $language_manager->getLanguages();

  $current_langcode = $language_manager->getCurrentLanguage()->getId();
  $language_links = [];
  foreach ($languages as $langcode => $language) {
    if ($node->hasTranslation($langcode)) {
      $translation = $node->getTranslation($langcode);
      $url = $translation->toUrl('canonical', ['language' => $language]);

      $language_links[$langcode] = [
        'title' => $language->getName(),
        'url' => $url->toString(),
        'active' => $langcode === $current_langcode,
        'langcode' => $langcode,
        'hreflang' => $langcode,
      ];
    }
  }

  $variables['language_links'] = $language_links;
  $variables['#cache']['contexts'][] = 'languages:language_interface';
}
```

---

## Contextual Filter par Langue

```yaml
# Views avec contextual filter "langcode" depuis l'URL
arguments:
  langcode:
    id: langcode
    table: node_field_data
    field: langcode
    default_action: default
    default_argument_type: language
    default_argument_options: {}   # Utilise la langue courante par défaut
```

---

## Debugging des Résultats Multilingues

```bash
# Vérifier quelle langue est détectée sur une page
docker compose exec php drush php:eval "
echo 'Langue courante: ' . \Drupal::languageManager()->getCurrentLanguage()->getId() . PHP_EOL;
echo 'Langue contenu: ' . \Drupal::languageManager()->getCurrentLanguage(\Drupal\Core\Language\LanguageInterface::TYPE_CONTENT)->getId() . PHP_EOL;
"

# Voir les résultats d'une View avec la langue comme argument
docker compose exec php drush php:eval "
\$view = \Drupal\views\Views::getView('articles_multilingue');
\$view->setDisplay('page_1');
\$view->setArguments(['fr']);
\$view->execute();
echo 'Résultats: ' . count(\$view->result) . PHP_EOL;
foreach (\$view->result as \$row) {
  echo \$row->node_field_data_title . ' (' . \$row->node_field_data_langcode . ')' . PHP_EOL;
}
"

# Vérifier le SQL généré par la View
function mon_module_views_query_alter(\Drupal\views\ViewExecutable \$view, \$query): void {
  if (\$view->id() === 'articles_multilingue') {
    \Drupal::logger('debug')->debug((string) \$view->query->query);
  }
}
```

---

## SEO Multilingue — hreflang

```php
// Ajouter les tags hreflang dans le <head> pour SEO
function mon_theme_preprocess_html(array &$variables): void {
  $node = \Drupal::routeMatch()->getParameter('node');
  if (!$node instanceof \Drupal\node\NodeInterface) {
    return;
  }

  $languages = \Drupal::languageManager()->getLanguages();
  foreach ($languages as $langcode => $language) {
    if ($node->hasTranslation($langcode)) {
      $translation = $node->getTranslation($langcode);
      $url = $translation->toUrl('canonical', [
        'language' => $language,
        'absolute' => TRUE,
      ]);

      $variables['#attached']['html_head'][] = [
        [
          '#tag' => 'link',
          '#attributes' => [
            'rel' => 'alternate',
            'hreflang' => $langcode,
            'href' => $url->toString(),
          ],
        ],
        'hreflang-' . $langcode,
      ];
    }
  }

  // x-default (langue par défaut)
  $default_lang = \Drupal::languageManager()->getDefaultLanguage();
  if ($node->hasTranslation($default_lang->getId())) {
    $url = $node->getTranslation($default_lang->getId())
      ->toUrl('canonical', ['absolute' => TRUE]);
    $variables['#attached']['html_head'][] = [
      [
        '#tag' => 'link',
        '#attributes' => [
          'rel' => 'alternate',
          'hreflang' => 'x-default',
          'href' => $url->toString(),
        ],
      ],
      'hreflang-x-default',
    ];
  }
}
```
