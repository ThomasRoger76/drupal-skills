---
name: drupal-content-modeling — custom entities
description: Créer des entités custom Drupal (ContentEntityBase) avec bundles, champs, AccessControlHandler, ListBuilder, et Forms. Quand choisir une custom entity vs un Node.
---

# Custom Content Entities — Référence Complète

## Quand Créer une Custom Entity

```
Node (Content Type) si :
  ✅ Contenu avec URL propre et SEO
  ✅ Révisions nécessaires (historique des modifications)
  ✅ Workflow éditorial (draft/published/archived)
  ✅ Traduction multilingue native
  ✅ Menu links et breadcrumbs automatiques

Custom Entity si :
  ✅ Données métier sans nécessité de frontend/SEO (commandes, réservations)
  ✅ Pas de révisions nécessaires
  ✅ Structure très différente d'un Node
  ✅ Relations complexes entre entités métier
  ✅ Performance : éviter la surcharge Node (path, revision, menu...)
  ✅ Données haute fréquence (logs, événements)
```

---

## Structure d'une Custom Entity

```
web/modules/custom/mon_module/
├── mon_module.info.yml
├── mon_module.routing.yml
├── mon_module.links.menu.yml
├── mon_module.links.action.yml
├── mon_module.permissions.yml
├── src/
│   └── Entity/
│       ├── Commande.php           ← ContentEntityBase
│       ├── CommandeAccessControlHandler.php
│       ├── CommandeInterface.php
│       ├── CommandeListBuilder.php
│       └── Form/
│           ├── CommandeForm.php
│           └── CommandeDeleteForm.php
└── templates/
    └── commande.html.twig
```

---

## ContentEntityBase — Implémentation

```php
<?php
// src/Entity/Commande.php
namespace Drupal\mon_module\Entity;

use Drupal\Core\Entity\ContentEntityBase;
use Drupal\Core\Entity\ContentEntityInterface;
use Drupal\Core\Entity\EntityTypeInterface;
use Drupal\Core\Field\BaseFieldDefinition;

/**
 * Entité Commande.
 *
 * D11+ — Remplacer l'annotation @ContentEntityType par un PHP Attribute :
 *
 * #[ContentEntityType(
 *   id: 'commande',
 *   label: new TranslatableMarkup('Commande'),
 *   label_collection: new TranslatableMarkup('Commandes'),
 *   handlers: [
 *     'access' => 'Drupal\mon_module\Entity\CommandeAccessControlHandler',
 *     'list_builder' => 'Drupal\mon_module\Entity\CommandeListBuilder',
 *     'form' => [
 *       'default' => 'Drupal\mon_module\Entity\Form\CommandeForm',
 *       'delete' => 'Drupal\mon_module\Entity\Form\CommandeDeleteForm',
 *     ],
 *     'route_provider' => [
 *       'html' => 'Drupal\Core\Entity\Routing\AdminHtmlRouteProvider',
 *     ],
 *   ],
 *   base_table: 'commande',
 *   translatable: true,
 *   admin_permission: 'administer commandes',
 *   entity_keys: [
 *     'id' => 'id', 'uuid' => 'uuid', 'langcode' => 'langcode', 'label' => 'reference',
 *   ],
 *   links: [
 *     'canonical' => '/admin/commandes/{commande}',
 *     'add-form' => '/admin/commandes/add',
 *     'edit-form' => '/admin/commandes/{commande}/edit',
 *     'delete-form' => '/admin/commandes/{commande}/delete',
 *     'collection' => '/admin/commandes',
 *   ],
 * )]
 *
 * D8-D10 — Garder l'annotation @ContentEntityType ci-dessous.
 * @ContentEntityType(
 *   id = "commande",
 *   label = @Translation("Commande"),
 *   label_collection = @Translation("Commandes"),
 *   label_singular = @Translation("commande"),
 *   label_plural = @Translation("commandes"),
 *   label_count = @PluralTranslation(
 *     singular = "@count commande",
 *     plural = "@count commandes",
 *   ),
 *   handlers = {
 *     "access" = "Drupal\mon_module\Entity\CommandeAccessControlHandler",
 *     "list_builder" = "Drupal\mon_module\Entity\CommandeListBuilder",
 *     "form" = {
 *       "default" = "Drupal\mon_module\Entity\Form\CommandeForm",
 *       "add" = "Drupal\mon_module\Entity\Form\CommandeForm",
 *       "edit" = "Drupal\mon_module\Entity\Form\CommandeForm",
 *       "delete" = "Drupal\mon_module\Entity\Form\CommandeDeleteForm",
 *     },
 *     "route_provider" = {
 *       "html" = "Drupal\Core\Entity\Routing\AdminHtmlRouteProvider",
 *     },
 *   },
 *   base_table = "commande",
 *   data_table = "commande_field_data",
 *   translatable = true,         ← TRUE si multilingue
 *   admin_permission = "administer commandes",
 *   entity_keys = {
 *     "id" = "id",
 *     "uuid" = "uuid",
 *     "langcode" = "langcode",
 *     "label" = "reference",    ← Champ utilisé comme label
 *   },
 *   links = {
 *     "canonical" = "/admin/commandes/{commande}",
 *     "add-form" = "/admin/commandes/add",
 *     "edit-form" = "/admin/commandes/{commande}/edit",
 *     "delete-form" = "/admin/commandes/{commande}/delete",
 *     "collection" = "/admin/commandes",
 *   },
 * )
 */
// D11 : utiliser #[ContentEntityType(...)] attribute PHP

class Commande extends ContentEntityBase implements ContentEntityInterface {

  /**
   * Définir les champs de base (BaseFieldDefinition).
   * Ces champs sont définis dans le code — pas dans la config.
   */
  public static function baseFieldDefinitions(EntityTypeInterface $entity_type): array {
    $fields = parent::baseFieldDefinitions($entity_type);  // id, uuid, langcode

    $fields['reference'] = BaseFieldDefinition::create('string')
      ->setLabel(t('Référence'))
      ->setDescription(t('Numéro de référence de la commande.'))
      ->setRequired(TRUE)
      ->setTranslatable(FALSE)      // ← Même référence dans toutes les langues
      ->setSettings([
        'max_length' => 64,
        'text_processing' => 0,
      ])
      ->setDisplayOptions('view', [
        'label' => 'above',
        'type' => 'string',
        'weight' => -10,
      ])
      ->setDisplayOptions('form', [
        'type' => 'string_textfield',
        'weight' => -10,
      ])
      ->setDisplayConfigurable('view', TRUE)
      ->setDisplayConfigurable('form', TRUE);

    $fields['statut'] = BaseFieldDefinition::create('list_string')
      ->setLabel(t('Statut'))
      ->setRequired(TRUE)
      ->setTranslatable(FALSE)
      ->setSetting('allowed_values', [
        'pending'   => t('En attente'),
        'confirmed' => t('Confirmée'),
        'shipped'   => t('Expédiée'),
        'cancelled' => t('Annulée'),
      ])
      ->setDefaultValue('pending')
      ->setDisplayOptions('view', ['type' => 'list_default', 'weight' => -9])
      ->setDisplayOptions('form', ['type' => 'options_select', 'weight' => -9])
      ->setDisplayConfigurable('view', TRUE)
      ->setDisplayConfigurable('form', TRUE);

    $fields['montant'] = BaseFieldDefinition::create('decimal')
      ->setLabel(t('Montant'))
      ->setDescription(t('Montant total en euros.'))
      ->setRequired(TRUE)
      ->setTranslatable(FALSE)
      ->setSetting('precision', 10)
      ->setSetting('scale', 2)
      ->setDisplayOptions('view', ['type' => 'number_decimal', 'weight' => -8])
      ->setDisplayOptions('form', ['type' => 'number', 'weight' => -8]);

    $fields['uid'] = BaseFieldDefinition::create('entity_reference')
      ->setLabel(t('Client'))
      ->setDescription(t('Utilisateur qui a passé la commande.'))
      ->setSetting('target_type', 'user')
      ->setSetting('handler', 'default')
      ->setTranslatable(FALSE)
      ->setDisplayOptions('view', ['type' => 'entity_reference_label', 'weight' => -7])
      ->setDisplayOptions('form', ['type' => 'entity_reference_autocomplete', 'weight' => -7]);

    $fields['notes'] = BaseFieldDefinition::create('text_long')
      ->setLabel(t('Notes internes'))
      ->setTranslatable(TRUE)     // ← Traduit par langue
      ->setDisplayOptions('form', ['type' => 'text_textarea', 'weight' => 0])
      ->setDisplayConfigurable('form', TRUE);

    $fields['created'] = BaseFieldDefinition::create('created')
      ->setLabel(t('Date de création'));

    $fields['changed'] = BaseFieldDefinition::create('changed')
      ->setLabel(t('Dernière modification'));

    return $fields;
  }

  // ── Méthodes d'accès aux champs ─────────────────────────────────────
  public function getReference(): string {
    return $this->get('reference')->value;
  }

  public function getStatut(): string {
    return $this->get('statut')->value;
  }

  public function getMontant(): float {
    return (float) $this->get('montant')->value;
  }
}
```

---

## AccessControlHandler

```php
<?php
// src/Entity/CommandeAccessControlHandler.php
namespace Drupal\mon_module\Entity;

use Drupal\Core\Access\AccessResult;
use Drupal\Core\Entity\EntityAccessControlHandler;
use Drupal\Core\Entity\EntityInterface;
use Drupal\Core\Session\AccountInterface;

class CommandeAccessControlHandler extends EntityAccessControlHandler {

  protected function checkAccess(EntityInterface $entity, $operation, AccountInterface $account): AccessResult {
    switch ($operation) {
      case 'view':
        // L'utilisateur peut voir ses propres commandes
        if ($entity->get('uid')->target_id === $account->id()) {
          return AccessResult::allowedIfHasPermission($account, 'view own commandes')
            ->cachePerUser()
            ->addCacheableDependency($entity);
        }
        return AccessResult::allowedIfHasPermission($account, 'view any commande')
          ->cachePerPermissions();

      case 'update':
        return AccessResult::allowedIfHasPermission($account, 'edit commandes')
          ->cachePerPermissions();

      case 'delete':
        return AccessResult::allowedIfHasPermission($account, 'delete commandes')
          ->cachePerPermissions();
    }

    return AccessResult::neutral()->cachePerPermissions();
  }

  protected function checkCreateAccess(AccountInterface $account, array $context, $entity_bundle = NULL): AccessResult {
    return AccessResult::allowedIfHasPermission($account, 'create commandes')
      ->cachePerPermissions();
  }
}
```

---

## Création Programmatique

```php
use Drupal\mon_module\Entity\Commande;

// Créer une commande
$commande = Commande::create([
  'reference' => 'CMD-' . date('Ymd') . '-' . rand(1000, 9999),
  'statut' => 'pending',
  'montant' => 149.99,
  'uid' => \Drupal::currentUser()->id(),
]);
$commande->save();

// Charger et modifier
$commande = \Drupal::entityTypeManager()->getStorage('commande')->load($id);
$commande->set('statut', 'confirmed');
$commande->save();

// Requêter
$commandes_ids = \Drupal::entityQuery('commande')
  ->condition('uid', $user_id)
  ->condition('statut', 'pending')
  ->sort('created', 'DESC')
  ->accessCheck(TRUE)
  ->execute();

$commandes = Commande::loadMultiple($commandes_ids);

// Supprimer
$commande->delete();
// OU supprimer plusieurs
\Drupal::entityTypeManager()->getStorage('commande')->delete($commandes);
```
