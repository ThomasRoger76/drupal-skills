---
name: drupal-api — graphql
description: Implémentation GraphQL avec le module drupal/graphql v4 pour Drupal - SchemaPlugin, ResolverBuilder, queries, mutations, et comparaison avec JSON:API.
---

# GraphQL pour Drupal — Référence Complète

## Quand Choisir GraphQL vs JSON:API

```
JSON:API (recommandé par défaut) :
  ✅ API standard sans configuration
  ✅ Filtres, includes, sparse fieldsets natifs
  ✅ Mutations (POST/PATCH/DELETE) sans code
  ✅ Cache Drupal natif
  ✅ Simple OAuth intégré
  → Idéal pour la majorité des projets headless

GraphQL :
  ✅ Requêtes complexes avec plusieurs entités en une seule request
  ✅ Pas d'over-fetching (demander exactement les champs nécessaires)
  ✅ Schema typé (meilleure DX côté frontend)
  ✅ Subscriptions (temps réel)
  → Idéal pour les frontends complexes avec beaucoup de relations
  → Projets avec équipe frontend qui préfère GraphQL
```

---

## Installation

```bash
composer require drupal/graphql
drush en graphql -y

# Version recommandée : drupal/graphql ^4
# GraphQL 4 utilise des Plugins PHP — pas de YAML schema
```

---

## Architecture GraphQL v4

```
SchemaPlugin   → Définit la structure du schéma GraphQL
ResolverBuilder → Construit les résolveurs pour chaque champ
DataProducer   → Produit les données (charge les entités, champs...)
```

---

## Créer un Schema GraphQL

```php
<?php
// src/Plugin/GraphQL/Schema/ArticlesSchema.php
namespace Drupal\mon_module\Plugin\GraphQL\Schema;

use Drupal\graphql\Plugin\GraphQL\Schema\SdlSchemaPluginBase;

/**
 * @Schema(
 *   id = "articles",
 *   name = "Articles Schema",
 *   description = "Schema GraphQL pour les articles du site."
 * )
 */
class ArticlesSchema extends SdlSchemaPluginBase {

  /**
   * Le schéma GraphQL en SDL (Schema Definition Language).
   */
  protected function getSchemaDefinition(): string {
    return <<<GQL
      type Query {
        article(id: ID!): Article
        articles(limit: Int = 10, offset: Int = 0, langcode: String): ArticleConnection!
        articleBySlug(slug: String!): Article
      }

      type Article {
        id: ID!
        uuid: String!
        title: String!
        body: String
        summary: String
        url: String!
        langcode: String!
        created: String!
        changed: String!
        author: User
        tags: [TaxonomyTerm!]!
        image: MediaImage
        status: Boolean!
      }

      type ArticleConnection {
        total: Int!
        items: [Article!]!
      }

      type User {
        id: ID!
        name: String!
        mail: String
      }

      type TaxonomyTerm {
        id: ID!
        name: String!
        vocabulary: String!
        url: String!
      }

      type MediaImage {
        id: ID!
        url: String!
        alt: String
        width: Int
        height: Int
      }

      type Mutation {
        updateArticleStatus(id: ID!, status: Boolean!): Article
      }
    GQL;
  }
}
```

---

## ResolverBuilder — Résolveurs

```php
<?php
// src/Plugin/GraphQL/Schema/ArticlesSchema.php (suite)

use Drupal\graphql\GraphQL\ResolverBuilder;
use Drupal\graphql\GraphQL\ResolverRegistry;

class ArticlesSchema extends SdlSchemaPluginBase {

  // ... schéma SDL ci-dessus ...

  /**
   * Enregistrer les résolveurs pour chaque champ du schéma.
   */
  public function getResolverRegistry(): ResolverRegistry {
    $builder = new ResolverBuilder();
    $registry = new ResolverRegistry();

    // ── Query.article ──────────────────────────────────────────────────
    $registry->addFieldResolver('Query', 'article',
      $builder->produce('entity_load')
        ->map('type', $builder->fromValue('node'))
        ->map('bundles', $builder->fromValue(['article']))
        ->map('id', $builder->fromArgument('id'))
        ->map('access', $builder->fromValue(TRUE))
    );

    // ── Query.articles ─────────────────────────────────────────────────
    $registry->addFieldResolver('Query', 'articles',
      $builder->produce('mon_module_articles_list')
        ->map('limit', $builder->fromArgument('limit'))
        ->map('offset', $builder->fromArgument('offset'))
        ->map('langcode', $builder->fromArgument('langcode'))
    );

    // ── Article.title ──────────────────────────────────────────────────
    $registry->addFieldResolver('Article', 'title',
      $builder->produce('entity_label')
        ->map('entity', $builder->fromParent())
    );

    // ── Article.body ───────────────────────────────────────────────────
    $registry->addFieldResolver('Article', 'body',
      $builder->produce('property_path')
        ->map('type', $builder->fromValue('entity:node'))
        ->map('value', $builder->fromParent())
        ->map('path', $builder->fromValue('body.processed'))
    );

    // ── Article.url ────────────────────────────────────────────────────
    $registry->addFieldResolver('Article', 'url',
      $builder->produce('entity_url')
        ->map('entity', $builder->fromParent())
    );

    // ── Article.author ─────────────────────────────────────────────────
    $registry->addFieldResolver('Article', 'author',
      $builder->produce('entity_owner')
        ->map('entity', $builder->fromParent())
    );

    // ── Article.tags ───────────────────────────────────────────────────
    $registry->addFieldResolver('Article', 'tags',
      $builder->produce('entity_reference')
        ->map('entity', $builder->fromParent())
        ->map('field', $builder->fromValue('field_tags'))
    );

    // ── User.name ──────────────────────────────────────────────────────
    $registry->addFieldResolver('User', 'name',
      $builder->produce('entity_label')
        ->map('entity', $builder->fromParent())
    );

    return $registry;
  }
}
```

---

## DataProducer Custom

```php
<?php
// src/Plugin/GraphQL/DataProducer/ArticlesList.php
namespace Drupal\mon_module\Plugin\GraphQL\DataProducer;

use Drupal\graphql\Plugin\GraphQL\DataProducer\DataProducerPluginBase;
use Drupal\node\Entity\Node;

/**
 * @DataProducer(
 *   id = "mon_module_articles_list",
 *   name = @Translation("Liste d'articles"),
 *   description = @Translation("Retourne une liste paginée d'articles."),
 *   produces = @ContextDefinition("any", label = @Translation("Articles")),
 *   consumes = {
 *     "limit" = @ContextDefinition("integer", label = @Translation("Limite"), required = FALSE),
 *     "offset" = @ContextDefinition("integer", label = @Translation("Offset"), required = FALSE),
 *     "langcode" = @ContextDefinition("string", label = @Translation("Langue"), required = FALSE),
 *   }
 * )
 */
class ArticlesList extends DataProducerPluginBase {

  public function resolve(
    int $limit = 10,
    int $offset = 0,
    ?string $langcode = NULL,
  ): array {
    $langcode = $langcode ?? \Drupal::languageManager()->getCurrentLanguage()->getId();

    $query = \Drupal::entityQuery('node')
      ->condition('type', 'article')
      ->condition('status', 1)
      ->condition('langcode', $langcode)
      ->sort('created', 'DESC')
      ->range($offset, $limit)
      ->accessCheck(TRUE);

    $total_query = clone $query;
    $total = $total_query->count()->execute();

    $nids = $query->execute();
    $nodes = Node::loadMultiple($nids);

    return [
      'total' => (int) $total,
      'items' => array_values($nodes),
    ];
  }
}
```

---

## Queries GraphQL — Exemples

```graphql
# Récupérer un article avec ses tags et auteur
query GetArticle($id: ID!) {
  article(id: $id) {
    id
    title
    body
    url
    created
    author {
      name
    }
    tags {
      name
      url
    }
    image {
      url
      alt
    }
  }
}

# Liste paginée d'articles en FR
query GetArticles($limit: Int, $offset: Int) {
  articles(limit: $limit, offset: $offset, langcode: "fr") {
    total
    items {
      id
      title
      summary
      url
      created
    }
  }
}
```

---

## Configuration du Schema

```yaml
# config/install/graphql.graphql_servers.articles_server.yml
langcode: fr
status: true
id: articles_server
label: 'Articles Server'
schema: articles                    # ID du SchemaPlugin
endpoint: /graphql/articles         # URL de l'endpoint
batching: true                      # Activer le batching des requêtes
caching: true                       # Cache basé sur les tags Drupal
debug_flag: false                   # Activer en dev pour voir les erreurs
```

---

## Sécuriser l'API GraphQL

```php
// Désactiver l'introspection en production
// settings.php
$config['graphql.graphql_servers.articles_server']['debug_flag'] = FALSE;

// Activer les persisted queries (recommandé en prod)
// → Seules les queries pré-approuvées peuvent être exécutées
```

```bash
# Tester l'endpoint GraphQL
curl -X POST https://mon-site.com/graphql/articles \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer TOKEN" \
  -d '{"query": "{ articles(limit: 3) { total items { title } } }"}'
```
