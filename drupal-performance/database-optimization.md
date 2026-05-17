---
name: drupal-performance — database optimization
description: Optimisation des requêtes base de données Drupal - slow query log, EXPLAIN, N+1 queries, indexes custom, cache des entités, et optimisation EntityQuery.
---

# Optimisation Base de Données — Référence Complète

## Activer le Slow Query Log (MariaDB / MySQL)

```ini
# /etc/mysql/conf.d/slow-queries.cnf (ou dans docker-compose.yml)
[mysqld]
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow-queries.log
long_query_time = 1          # Logguer les requêtes > 1 seconde
log_queries_not_using_indexes = 1    # Logguer aussi les requêtes sans index
```

```bash
# Dans Docker Compose (MariaDB) :
command: >
  --slow-query-log=1
  --slow-query-log-file=/var/lib/mysql/slow-queries.log
  --long-query-time=1
  --log-queries-not-using-indexes=1

# Analyser le slow query log
docker compose exec database tail -100 /var/lib/mysql/slow-queries.log
```

---

## EXPLAIN — Analyser une Requête Lente

```bash
# Depuis Drush
drush sql-query "EXPLAIN SELECT nfd.nid, nfd.title FROM node_field_data nfd WHERE nfd.type = 'article' AND nfd.status = 1 ORDER BY nfd.created DESC LIMIT 10"

# Interpréter EXPLAIN :
# type = 'ALL' → Full table scan → INDEX manquant
# type = 'ref' → Utilise un index → Bien
# type = 'eq_ref' → Utilise la clé primaire → Parfait
# rows = 50000 → Trop de lignes scannées → Problème
# Extra = 'Using filesort' → Tri en mémoire → Potentiellement lent
```

---

## N+1 Queries — Identifier et Corriger

### Le Problème

```php
// ❌ N+1 QUERIES — 1 requête pour les nœuds + N requêtes pour les auteurs
$nids = \Drupal::entityQuery('node')
  ->condition('type', 'article')
  ->condition('status', 1)
  ->execute();

$nodes = Node::loadMultiple($nids);

foreach ($nodes as $node) {
  $uid = $node->getOwnerId();
  $user = User::load($uid);           // ← 1 requête par nœud !
  echo $user->getDisplayName();
}
// → Pour 50 articles : 1 + 50 = 51 requêtes
```

### Solutions

```php
// ✅ Solution 1 : loadMultiple en une requête
$nids = \Drupal::entityQuery('node')
  ->condition('type', 'article')
  ->condition('status', 1)
  ->execute();

$nodes = Node::loadMultiple($nids);

// Collecter tous les UIDs en une passe
$uids = array_unique(array_map(fn($n) => $n->getOwnerId(), $nodes));

// Charger tous les utilisateurs en UNE requête
$users = User::loadMultiple($uids);

foreach ($nodes as $node) {
  $user = $users[$node->getOwnerId()] ?? NULL;
  if ($user) {
    echo $user->getDisplayName();
  }
}
// → 2 requêtes total

// ✅ Solution 2 : Views avec JOIN (recommandé pour les listes)
// Views::getView() avec JOIN sur users_field_data → 1 seule requête SQL

// ✅ Solution 3 : Database API avec JOIN direct
$result = \Drupal::database()->select('node_field_data', 'nfd')
  ->fields('nfd', ['nid', 'title', 'uid'])
  ->fields('ufd', ['name', 'mail'])
  ->join('users_field_data', 'ufd', 'nfd.uid = ufd.uid')
  ->condition('nfd.type', 'article')
  ->condition('nfd.status', 1)
  ->orderBy('nfd.created', 'DESC')
  ->range(0, 10)
  ->execute()
  ->fetchAll();
```

### N+1 avec Paragraphs

```php
// ❌ PROBLÈME : chargement un par un
$node = Node::load($nid);
foreach ($node->get('field_contenu') as $item) {
  $paragraph = $item->entity;       // 1 requête par paragraphe
  echo $paragraph->getType();
}

// ✅ SOLUTION : batch loading
$node = Node::load($nid);
$ids = array_column($node->get('field_contenu')->getValue(), 'target_id');
$paragraphs = \Drupal::entityTypeManager()
  ->getStorage('paragraph')
  ->loadMultiple($ids);             // 1 seule requête
```

---

## Ajouter des Index sur les Tables Custom

```php
// Dans hook_schema() — définir les index
function mon_module_schema(): array {
  return [
    'mon_module_commandes' => [
      'description' => 'Table des commandes.',
      'fields' => [
        'id' => ['type' => 'serial', 'not null' => TRUE],
        'uid' => ['type' => 'int', 'not null' => TRUE, 'default' => 0],
        'nid' => ['type' => 'int', 'not null' => TRUE, 'default' => 0],
        'statut' => ['type' => 'varchar', 'length' => 32, 'not null' => TRUE, 'default' => 'pending'],
        'montant' => ['type' => 'int', 'not null' => TRUE, 'default' => 0],
        'created' => ['type' => 'int', 'not null' => TRUE, 'default' => 0],
      ],
      'primary key' => ['id'],
      'indexes' => [
        'uid' => ['uid'],                          // Chercher par utilisateur
        'statut' => ['statut'],                    // Filtrer par statut
        'uid_statut' => ['uid', 'statut'],         // Index composite
        'created' => ['created'],                  // Trier par date
        'nid_statut' => ['nid', 'statut'],         // Commandes d'un produit
      ],
    ],
  ];
}

// Ajouter un index sur une table existante via hook_update_N
function mon_module_update_9001(): void {
  $schema = \Drupal::database()->schema();

  if (!$schema->indexExists('mon_module_commandes', 'uid_statut')) {
    $schema->addIndex(
      'mon_module_commandes',
      'uid_statut',
      ['uid', 'statut'],
      [
        'fields' => [
          'uid' => ['type' => 'int', 'not null' => TRUE, 'default' => 0],
          'statut' => ['type' => 'varchar', 'length' => 32, 'not null' => TRUE],
        ],
      ]
    );
  }
}
```

---

## EntityQuery Optimisée

```php
// ❌ EntityQuery sans accessCheck (deprecated D9+, warning D10, fatal D11)
$query = \Drupal::entityQuery('node');

// ✅ Toujours spécifier accessCheck
$query = \Drupal::entityQuery('node')
  ->accessCheck(TRUE)     // Vérifier les permissions (recommandé)
  ->accessCheck(FALSE);   // Ignorer les permissions (scripts cron, migrations)

// ❌ Charger toutes les entités pour compter
$count = count(Node::loadMultiple(
  \Drupal::entityQuery('node')->execute()
));

// ✅ count() sans charger les entités
$count = \Drupal::entityQuery('node')
  ->condition('type', 'article')
  ->condition('status', 1)
  ->accessCheck(TRUE)
  ->count()              // ← ne retourne qu'un entier, pas les IDs
  ->execute();

// ✅ Pagination avec range()
$nids = \Drupal::entityQuery('node')
  ->condition('type', 'article')
  ->condition('status', 1)
  ->sort('created', 'DESC')
  ->range(0, 10)          // ← LIMIT 10 OFFSET 0
  ->accessCheck(TRUE)
  ->execute();

// ✅ Conditions sur les champs entity reference
$nids = \Drupal::entityQuery('node')
  ->condition('field_tags.entity.name', 'Drupal')   // Condition sur la relation
  ->accessCheck(TRUE)
  ->execute();
```

---

## Compter les Requêtes Drupal (Debug)

```php
// Activer le logging des requêtes (développement uniquement)
// Dans settings.local.php :
$settings['devel.settings']['query_display'] = TRUE;

// Depuis PHP — compter les requêtes sur une page
function mon_module_get_query_count(): int {
  $connection = \Drupal::database();
  if ($connection->getLogger()) {
    return count($connection->getLogger()->get('default'));
  }
  return 0;
}

// Afficher toutes les requêtes avec Devel module
// → /devel/query-log (si activé dans les settings Devel)
```

---

## Optimisation de Base — Checklist

```bash
# 1. Vérifier l'état des tables (fragmentation)
drush sql-query "SHOW TABLE STATUS LIKE 'node_field_data'"

# 2. Analyser les index utilisés
drush sql-query "SHOW INDEX FROM node_field_data"

# 3. Optimiser les tables fragmentées
drush sql-query "OPTIMIZE TABLE node_field_data"

# 4. Vérifier la taille des tables (tables trop lourdes)
drush sql-query "SELECT table_name, ROUND(data_length/1024/1024, 2) AS 'Data MB', ROUND(index_length/1024/1024, 2) AS 'Index MB' FROM information_schema.tables WHERE table_schema = DATABASE() ORDER BY data_length DESC LIMIT 20"

# 5. Tables watchdog et cache très volumineuses → vider régulièrement
drush sql-query "TRUNCATE watchdog"
drush cr
```
