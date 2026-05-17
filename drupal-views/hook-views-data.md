---
name: drupal-views — hook_views_data
description: Exposer des tables DB custom ou des données tierces à Views via hook_views_data() et hook_views_data_alter(). Structure complète des définitions, relations, handlers, et leur intégration dans le plugin system Views.
---

# hook_views_data() — Exposer des Données à Views

## Vue d'ensemble

`hook_views_data()` est le point d'entrée principal pour rendre n'importe quelle table disponible dans Views. Chaque clé de tableau correspond à un alias de table, chaque sous-clé à un champ exposé avec ses handlers.

**Après chaque modification :** `drush cr` — Views découvre les tables via le cache.

---

## Structure de Base

```php
<?php
// mon_module/mon_module.module (ou mon_module.views.inc — include dans .module)

/**
 * Implements hook_views_data().
 */
function mon_module_views_data(): array {
  $data = [];

  // ── Définir la table principale ────────────────────────────────────────────
  $data['mon_module_commandes'] = [];

  // En-tête de table — metadata obligatoire
  $data['mon_module_commandes']['table'] = [
    'group' => t('Mon Module — Commandes'),   // Groupe dans l'UI Views
    'base' => [
      'field' => 'id',                         // Clé primaire
      'title' => t('Commandes'),
      'help' => t('Toutes les commandes du site.'),
      'weight' => -10,                         // Position dans la liste des tables base
    ],
    // 'join' si cette table n'est PAS une table de base (voir Relations ci-dessous)
  ];

  // ── Champs ─────────────────────────────────────────────────────────────────

  $data['mon_module_commandes']['id'] = [
    'title' => t('ID Commande'),
    'help' => t('Identifiant unique de la commande.'),
    'field' => [
      'id' => 'numeric',                       // Handler field natif Drupal
    ],
    'filter' => [
      'id' => 'numeric',
    ],
    'sort' => [
      'id' => 'standard',
    ],
    'argument' => [                            // Contextual filter
      'id' => 'numeric',
    ],
  ];

  $data['mon_module_commandes']['reference'] = [
    'title' => t('Référence'),
    'help' => t('Numéro de référence de la commande.'),
    'field' => [
      'id' => 'standard',                      // String
    ],
    'filter' => [
      'id' => 'string',
    ],
    'sort' => [
      'id' => 'standard',
    ],
    'argument' => [
      'id' => 'string',
    ],
  ];

  $data['mon_module_commandes']['statut'] = [
    'title' => t('Statut'),
    'help' => t('Statut de la commande (pending, confirmed, shipped, cancelled).'),
    'field' => [
      'id' => 'standard',
    ],
    'filter' => [
      'id' => 'in_operator',                   // Filtre avec liste de valeurs
      'options callback' => 'mon_module_get_statuts',  // Callback qui retourne les options
      'options arguments' => [],
    ],
    'sort' => [
      'id' => 'standard',
    ],
  ];

  $data['mon_module_commandes']['montant'] = [
    'title' => t('Montant'),
    'help' => t('Montant total de la commande en centimes.'),
    'field' => [
      'id' => 'numeric',
      'float' => TRUE,
    ],
    'filter' => [
      'id' => 'numeric',
    ],
    'sort' => [
      'id' => 'standard',
    ],
  ];

  $data['mon_module_commandes']['created'] = [
    'title' => t('Date de création'),
    'help' => t('Timestamp UNIX de création de la commande.'),
    'field' => [
      'id' => 'date',                          // Formatage date automatique
    ],
    'filter' => [
      'id' => 'date',
    ],
    'sort' => [
      'id' => 'date',
    ],
  ];

  $data['mon_module_commandes']['uid'] = [
    'title' => t('Utilisateur'),
    'help' => t('UID de l\'utilisateur qui a passé la commande.'),
    'field' => [
      'id' => 'standard',
    ],
    'filter' => [
      'id' => 'numeric',
    ],
    'sort' => [
      'id' => 'standard',
    ],
    // Relation vers la table users_field_data
    'relationship' => [
      'id' => 'standard',
      'base' => 'users_field_data',            // Table cible
      'base field' => 'uid',                   // Champ de jointure dans la table cible
      'title' => t('Utilisateur de la commande'),
      'help' => t('Accéder aux informations de l\'utilisateur.'),
      'label' => t('Utilisateur'),
    ],
  ];

  $data['mon_module_commandes']['nid'] = [
    'title' => t('Produit associé'),
    'help' => t('NID du nœud produit lié à cette commande.'),
    'relationship' => [
      'id' => 'standard',
      'base' => 'node_field_data',
      'base field' => 'nid',
      'title' => t('Produit'),
      'label' => t('Produit'),
    ],
    'field' => ['id' => 'numeric'],
    'filter' => ['id' => 'numeric'],
  ];

  return $data;
}

/**
 * Callback pour les options du filtre 'statut'.
 */
function mon_module_get_statuts(): array {
  return [
    'pending' => t('En attente'),
    'confirmed' => t('Confirmée'),
    'shipped' => t('Expédiée'),
    'cancelled' => t('Annulée'),
  ];
}
```

---

## hook_views_data_alter() — Modifier des Données Existantes

```php
/**
 * Implements hook_views_data_alter().
 *
 * Ajouter des champs, relations ou handlers sur des tables que d'autres
 * modules ont déjà déclarées dans hook_views_data().
 */
function mon_module_views_data_alter(array &$data): void {
  // Ajouter une relation depuis node_field_data vers notre table custom
  if (isset($data['node_field_data'])) {
    $data['node_field_data']['commandes_du_noeud'] = [
      'title' => t('Commandes'),
      'help' => t('Commandes liées à ce nœud produit.'),
      'relationship' => [
        'id' => 'standard',
        'base' => 'mon_module_commandes',
        'base field' => 'nid',
        'field' => 'nid',                      // Champ dans node_field_data
        'title' => t('Commandes du produit'),
        'label' => t('Commandes'),
      ],
    ];
  }

  // Modifier un handler existant
  if (isset($data['node_field_data']['title'])) {
    // Remplacer le filtre par un filtre custom
    $data['node_field_data']['title']['filter']['id'] = 'mon_module_title_filter';
  }
}
```

---

## Référence des Handler IDs Natifs

### Fields
| Handler ID | Usage |
|-----------|-------|
| `standard` | Texte simple (string) |
| `numeric` | Entier / décimal |
| `date` | Timestamp UNIX → date formatée |
| `boolean` | 0/1 → Oui/Non |
| `url` | Lien cliquable |
| `markup` | HTML — rendu tel quel |
| `entity` | Charge et rend une entité |
| `rendered_entity` | Entité avec view mode |

### Filters
| Handler ID | Usage |
|-----------|-------|
| `string` | LIKE, IS EMPTY, IS NOT EMPTY |
| `numeric` | =, !=, <, >, BETWEEN |
| `date` | Date range, relatif, absolu |
| `boolean` | TRUE/FALSE |
| `in_operator` | SELECT, IN LIST |
| `bundle` | Par bundle d'entité |
| `language` | Par langcode |

### Sorts
| Handler ID | Usage |
|-----------|-------|
| `standard` | ORDER BY simple |
| `date` | Tri sur timestamp |
| `random` | ORDER BY RAND() |

### Arguments (Contextual Filters)
| Handler ID | Usage |
|-----------|-------|
| `string` | Argument texte |
| `numeric` | Argument numérique |
| `date_fulldate` | Date complète YYYYMMDD |
| `uid` | UID utilisateur |
| `nid` | NID node |
| `entity_id` | ID entité générique |

---

## Table Non-Base (Join uniquement)

```php
// Table qui n'est jamais une "base" de View — uniquement joignable via relation
$data['mon_module_meta']['table'] = [
  'group' => t('Meta commandes'),
  // PAS de clé 'base' → la table ne peut pas être le point de départ d'une View
  'join' => [
    'mon_module_commandes' => [              // Table parente
      'left_field' => 'id',                 // Champ dans mon_module_commandes
      'field' => 'commande_id',             // Champ dans mon_module_meta
      'type' => 'LEFT',                     // LEFT JOIN (ou INNER)
    ],
  ],
];

$data['mon_module_meta']['valeur'] = [
  'title' => t('Valeur méta'),
  'field' => ['id' => 'standard'],
  'filter' => ['id' => 'string'],
];
```

---

## Organisation du Code — hook_views_data dans un fichier séparé

```php
// mon_module.module — déclarer l'include
/**
 * Implements hook_views_data().
 */
function mon_module_views_data() {
  // Pour les modules avec beaucoup de données Views, externaliser dans un fichier
  // dédié chargé automatiquement par Drupal si déclaré dans le .info.yml
  // Non nécessaire depuis D8 — Drupal trouve views_data.inc automatiquement
  // si le fichier s'appelle mon_module.views.inc
}
```

Alternativement, créer `mon_module.views.inc` — Drupal l'inclut automatiquement :

```php
// mon_module/mon_module.views.inc
function mon_module_views_data(): array {
  // ... tout le code hook_views_data ici
}
```

---

## Bonnes Pratiques

```php
// Toujours inclure 'title' et 'help' pour chaque entrée
// — ils apparaissent dans l'UI Views pour aider les éditeurs

// Toujours appeler drush cr après modification
// — Views met en cache la découverte des tables

// Pour déboguer quels handlers sont disponibles :
// drush php:eval "var_dump(array_keys(\Drupal::moduleHandler()->invokeAll('views_data')));"

// Pour voir la définition d'une table spécifique :
// drush php:eval "\$v = \Drupal::service('views.views_data'); var_dump(\$v->get('mon_module_commandes'));"
```
