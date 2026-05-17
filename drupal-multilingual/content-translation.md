---
name: drupal-multilingual — content translation
description: Traduction de contenu Drupal (nodes, entités) - configuration des champs traduisibles, API PHP addTranslation/getTranslation, EntityQuery par langue, gestion du fallback, et traduction programmatique.
---

# Traduction de Contenu — Référence Complète

## Activer la Traduction d'un Content Type

```yaml
# config/install/language.content_settings.node.article.yml
langcode: fr
status: true
id: node.article
target_entity_type_id: node
target_bundle: article
default_langcode: site_default
language_alterable: true

# Rendre le type traduisible
third_party_settings:
  content_translation:
    enabled: true
    bundle_settings:
      untranslatable_fields_hide: true
```

**Via l'UI :** `/admin/config/regional/content-language` → cocher le Content Type et les champs à traduire.

**Règle :** certains champs ne doivent PAS être traduits (ex: `field_date`, `field_prix`, `field_statut`) — décocher "Translatable" dans la config du champ.

---

## API PHP — Traduction d'Entités

### Créer une Traduction

```php
<?php

use Drupal\node\Entity\Node;

// Charger le nœud en langue d'origine
$node = Node::load(42);

// Vérifier si la traduction existe déjà
if (!$node->hasTranslation('fr')) {
  // Créer la traduction française
  $translation = $node->addTranslation('fr', [
    'title' => 'Mon titre en français',
    'body' => [
      'value' => '<p>Contenu en français.</p>',
      'format' => 'basic_html',
    ],
    'field_resume' => 'Résumé en français',
    // Les champs NON traduisibles (field_date, etc.) sont ignorés
    // — ils gardent la valeur de la langue d'origine
  ]);

  // Optionnel : marquer comme traduit
  $translation->set('content_translation_status', TRUE);
  $translation->set('content_translation_uid', \Drupal::currentUser()->id());

  $node->save();
}
else {
  // Mettre à jour la traduction existante
  $translation = $node->getTranslation('fr');
  $translation->set('title', 'Titre mis à jour en français');
  $translation->save();   // ← save() sur la traduction, pas sur $node
}
```

### Charger une Traduction

```php
// Charger la traduction pour une langue spécifique
$node = Node::load(42);

if ($node->hasTranslation('fr')) {
  $fr_translation = $node->getTranslation('fr');
  $titre_fr = $fr_translation->getTitle();       // Titre FR
  $titre_orig = $node->getTitle();               // Titre langue d'origine
}

// ✅ Charger la traduction du CONTEXTE COURANT (recommandé)
// Prend en compte la langue de l'URL, de l'utilisateur, etc.
$node = \Drupal::entityTypeManager()->getStorage('node')->load(42);
$node_traduit = \Drupal::service('entity.repository')
  ->getTranslationFromContext($node);

// Vérifier la langue d'une traduction
$langcode = $node_traduit->language()->getId();  // 'fr', 'en', etc.
```

### Lister les Traductions d'un Nœud

```php
$node = Node::load(42);

// Obtenir tous les langcodes traduits
$languages = $node->getTranslationLanguages();
// → ['fr' => LanguageInterface, 'en' => LanguageInterface, 'de' => LanguageInterface]

foreach ($languages as $langcode => $language) {
  $translation = $node->getTranslation($langcode);
  echo $langcode . ': ' . $translation->getTitle() . "\n";
}

// Vérifier si une traduction est complète (pas juste créée)
$status = $node->getTranslation('fr')
  ->get('content_translation_status')
  ->value;
// 1 = traduit, 0 = en attente/ébauche
```

### Supprimer une Traduction

```php
$node = Node::load(42);
if ($node->hasTranslation('fr') && $node->language()->getId() !== 'fr') {
  $node->removeTranslation('fr');
  $node->save();
}
// ⚠️ Ne pas supprimer la langue d'origine (defaultTranslation)
```

---

## EntityQuery Multilingue

```php
// Chercher des nœuds en français publiés
$query = \Drupal::entityQuery('node')
  ->condition('status', 1)
  ->condition('langcode', 'fr')           // Uniquement les traductions FR
  ->condition('type', 'article')
  ->accessCheck(TRUE)
  ->sort('created', 'DESC');

$nids = $query->execute();
$nodes = Node::loadMultiple($nids);

// Obtenir les traductions FR des nœuds chargés
foreach ($nodes as $node) {
  if ($node->hasTranslation('fr')) {
    $fr_node = $node->getTranslation('fr');
    // Travailler avec la version FR
  }
}
```

**Important :** `->condition('langcode', 'fr')` filtre les nœuds qui ONT une traduction FR. Les nœuds sans traduction FR ne sont pas retournés.

---

## Generéer des URLs par Langue

```php
use Drupal\Core\Url;
use Drupal\language\Entity\ConfigurableLanguage;

// URL d'un nœud dans une langue spécifique
$node = Node::load(42);
$language_fr = ConfigurableLanguage::load('fr');

$url = Url::fromRoute(
  'entity.node.canonical',
  ['node' => $node->id()],
  ['language' => $language_fr]
);

$url_string = $url->toString();
// → /fr/mon-article (si préfixe URL activé)
```

---

## Preprocess Multilingue — Pattern Correct

```php
// ✅ CORRECT — charger la traduction du contexte courant
function mon_theme_preprocess_node(array &$variables): void {
  /** @var \Drupal\node\Entity\Node $node */
  $node = $variables['node'];

  // Sur un site multilingue, $node peut être dans la langue d'origine
  // même si on navigue en FR — getTranslationFromContext corrige ça
  $node = \Drupal::service('entity.repository')
    ->getTranslationFromContext($node);

  // Ajouter le cache context langue
  $variables['#cache']['contexts'][] = 'languages:language_content';

  // Accéder aux champs dans la langue courante
  $variables['titre_courant'] = $node->getTitle();
}

// ❌ INCORRECT — $node peut être dans la langue d'origine
function mon_theme_preprocess_node_mauvais(array &$variables): void {
  $node = $variables['node'];
  $variables['titre_courant'] = $node->getTitle(); // Toujours la langue source !
}
```

---

## Language Fallback — Configuration

```yaml
# Configuration du fallback si traduction manquante
# /admin/config/regional/content-language

# Via config :
# language.negotiation.yml → language_content
# language_content.settings.yml → fallback configuré

# Via PHP :
$language_manager = \Drupal::languageManager();
$fallback = $language_manager->getFallbackCandidates(
  \Drupal\Core\Language\LanguageInterface::TYPE_CONTENT
);
```

**Comportement par défaut :** si un nœud n'a pas de traduction FR, Drupal affiche la langue d'origine (EN) au lieu d'une 404.

Pour personnaliser : `hook_language_fallback_candidates_alter()`.

---

## Traduction des Champs Entity Reference

```php
// Un champ Entity Reference (ex: field_auteur, field_categorie)
// n'est PAS traduit par défaut — même valeur pour toutes les langues

// Si vous avez besoin de références différentes par langue :
// → Marquer le champ comme "Translatable" dans la config du champ
// → La relation diffère selon la langue du parent

// Exemple : champ taxonomy_term reference traduisible
$node = Node::load(42);
$fr_node = $node->getTranslation('fr');

// Les terms référencés dans la version FR
$terms_fr = $fr_node->get('field_categories')->referencedEntities();
```

---

## Commandes Drush Utiles

```bash
# Lister les langues du site
drush php:eval "foreach (\Drupal::languageManager()->getLanguages() as \$lang) { echo \$lang->getId() . ': ' . \$lang->getName() . PHP_EOL; }"

# Compter les traductions par langue
drush php:eval "
\$storage = \Drupal::entityTypeManager()->getStorage('node');
foreach (['fr', 'en', 'de'] as \$lang) {
  \$count = \$storage->getQuery()->condition('langcode', \$lang)->condition('status', 1)->accessCheck(FALSE)->count()->execute();
  echo \$lang . ': ' . \$count . ' noeuds traduits' . PHP_EOL;
}
"

# Créer une traduction en masse (script Drush)
drush php:script translate_batch.php
```
