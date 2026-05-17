---
name: drupal-search — custom processors
description: Créer des processors Search API custom pour transformer les données à l'indexation ou modifier les queries - DataAlterProcessor, QueryPreprocessor, PostprocessorInterface.
---

# Custom Search API Processors — Référence Complète

## Types de Processors

```
DataAlterProcessor   → Modifie les données AVANT l'indexation (index time)
QueryPreprocessor    → Modifie la query AVANT l'exécution (query time)
PostprocessorInterface → Modifie les résultats APRÈS l'exécution (result time)
FieldsProcessor      → Applique une transformation sur des champs spécifiques
```

---

## DataAlterProcessor — Transformer les Données à l'Indexation

```php
<?php
// src/Plugin/search_api/processor/PrixDeviseProcessor.php
namespace Drupal\mon_module\Plugin\search_api\processor;

use Drupal\search_api\Processor\ProcessorPluginBase;
use Drupal\search_api\Processor\ProcessorProperty;
use Drupal\search_api\Item\ItemInterface;
use Drupal\Core\Plugin\ContainerFactoryPluginInterface;

/**
 * Convertit les prix en centimes pour un tri uniforme.
 *
 * @SearchApiProcessor(
 *   id = "mon_module_prix_devise",
 *   label = @Translation("Prix en centimes"),
 *   description = @Translation("Convertit les prix décimaux en entiers pour le tri."),
 *   stages = {
 *     "add_properties" = 0,
 *     "preprocess_index" = 0,
 *   }
 * )
 */
class PrixDeviseProcessor extends ProcessorPluginBase {

  /**
   * Déclarer les propriétés ajoutées à l'index par ce processor.
   */
  public function getPropertyDefinitions(\Drupal\Core\TypedData\ComplexDataDefinitionInterface|null $datasource = NULL): array {
    $properties = [];

    if (!$datasource) {
      // Propriété calculée (pas liée à une datasource spécifique)
      $definition = \Drupal\Core\TypedData\DataDefinition::create('integer')
        ->setLabel($this->t('Prix en centimes'))
        ->setDescription($this->t('Prix converti en centimes pour le tri.'));

      $properties['prix_centimes'] = new ProcessorProperty($definition);
    }

    return $properties;
  }

  /**
   * Ajouter la valeur calculée pour chaque item indexé.
   */
  public function addFieldValues(ItemInterface $item): void {
    // Récupérer l'entité source
    $entity = $item->getOriginalObject()->getValue();

    if (!$entity instanceof \Drupal\commerce_product\Entity\ProductVariation) {
      return;
    }

    // Calculer le prix en centimes
    $price = $entity->get('price')->number;
    $prix_centimes = (int) round($price * 100);

    // Ajouter le champ calculé
    $fields = $item->getFields(FALSE);
    if (isset($fields['prix_centimes'])) {
      $fields['prix_centimes']->addValue($prix_centimes);
    }
  }
}
```

---

## QueryPreprocessor — Modifier la Query Search API

```php
<?php
// src/Plugin/search_api/processor/FiltreRoleProcessor.php
namespace Drupal\mon_module\Plugin\search_api\processor;

use Drupal\search_api\Processor\ProcessorPluginBase;
use Drupal\search_api\Query\QueryInterface;

/**
 * Filtre les résultats selon le rôle de l'utilisateur courant.
 *
 * @SearchApiProcessor(
 *   id = "mon_module_filtre_role",
 *   label = @Translation("Filtre par rôle utilisateur"),
 *   description = @Translation("Masque certains contenus selon les rôles."),
 *   stages = {
 *     "preprocess_query" = -10,
 *   }
 * )
 */
class FiltreRoleProcessor extends ProcessorPluginBase {

  /**
   * Modifier la query Search API avant son exécution.
   */
  public function preprocessSearchQuery(QueryInterface $query): void {
    $current_user = \Drupal::currentUser();

    // Les anonymes ne voient pas les contenus "membres seulement"
    if ($current_user->isAnonymous()) {
      $condition_group = $query->createConditionGroup('OR');
      $condition_group->addCondition('field_acces_restreint', NULL, '=');
      $condition_group->addCondition('field_acces_restreint', 0, '=');
      $query->addConditionGroup($condition_group);
    }

    // Ajouter un tag de cache pour la query
    $query->addTag('mon_module_filtre_role');

    // Modifier les métadonnées de cache
    $query->getSearchApiQuery()
      ?->getCacheContexts()
      ?->addCacheContext('user.roles');
  }
}
```

---

## Processor pour le hook_search_api_query_alter()

```php
/**
 * Implements hook_search_api_query_alter().
 *
 * Alternative aux processors — modifier toutes les queries Search API.
 * Plus simple pour des modifications globales.
 */
function mon_module_search_api_query_alter(\Drupal\search_api\Query\QueryInterface $query): void {
  $index = $query->getIndex();

  // Agir uniquement sur l'index "articles"
  if ($index->id() !== 'articles_index') {
    return;
  }

  // Exclure les nœuds archivés de tous les résultats
  $query->addCondition('field_statut_archive', FALSE);

  // Boost les contenus récents
  if ($query->getProcessingLevel() !== QueryInterface::PROCESSING_NONE) {
    $query->setOption('search_api_boosting', [
      'boosting_terms' => [
        'created' => ['boost' => 2.0, 'type' => 'recent'],
      ],
    ]);
  }
}
```

---

## FieldsProcessor — Transformation de Champs

```php
<?php
// src/Plugin/search_api/processor/NormaliserTexteProcessor.php

/**
 * Normalise le texte pour améliorer la recherche.
 *
 * @SearchApiProcessor(
 *   id = "mon_module_normaliser_texte",
 *   label = @Translation("Normaliser le texte"),
 *   description = @Translation("Supprime les accents et normalise le texte."),
 *   stages = {
 *     "preprocess_index" = -15,
 *     "preprocess_query" = -15,
 *   },
 *   locked = false,
 *   hidden = false,
 * )
 */
class NormaliserTexteProcessor extends \Drupal\search_api\Processor\FieldsProcessorPluginBase {

  /**
   * Traiter la valeur d'un champ.
   */
  protected function processFieldValue(string &$value, string $type): void {
    if ($type !== 'text') {
      return;
    }

    // Supprimer les accents (translitération)
    $transliterator = \Transliterator::createFromRules(
      ':: Any-Latin; :: Latin-ASCII; :: NFD; :: [:Nonspacing Mark:] Remove; :: Lower();'
    );

    if ($transliterator) {
      $value = $transliterator->transliterate($value);
    }

    // Supprimer la ponctuation excessive
    $value = preg_replace('/[^\w\s]/u', ' ', $value);
    $value = trim(preg_replace('/\s+/', ' ', $value));
  }

  /**
   * Ce processor s'applique-t-il à ce type de champ ?
   */
  protected function testType(string $type): bool {
    return in_array($type, ['text', 'string']);
  }
}
```

---

## Configurer un Processor Custom via YAML

```yaml
# Dans la config de l'index Search API
processor_settings:
  mon_module_prix_devise:
    processor_id: mon_module_prix_devise
    weights:
      add_properties: 0
      preprocess_index: 0
    settings: {}

  mon_module_filtre_role:
    processor_id: mon_module_filtre_role
    weights:
      preprocess_query: -10
    settings: {}

  mon_module_normaliser_texte:
    processor_id: mon_module_normaliser_texte
    weights:
      preprocess_index: -15
      preprocess_query: -15
    settings:
      all_fields: true
      fields:
        title: title
        body: body
```

---

## Tester un Processor Custom

```bash
# Vider l'index et réindexer (pour voir les changements de DataAlterProcessor)
drush sapi-c articles_index
drush sapi-i articles_index

# Tester la query avec le QueryPreprocessor
drush php:eval "
\$query = \Drupal::entityTypeManager()
  ->getStorage('search_api_index')
  ->load('articles_index')
  ->query();
\$query->keys('test');
\$result = \$query->execute();
echo 'Résultats après processor: ' . \$result->getResultCount() . PHP_EOL;
"

# Voir les processors actifs sur un index
drush php:eval "
\$index = \Drupal::entityTypeManager()->getStorage('search_api_index')->load('articles_index');
foreach (\$index->getProcessors() as \$id => \$processor) {
  echo \$id . ': ' . get_class(\$processor) . PHP_EOL;
}
"
```
