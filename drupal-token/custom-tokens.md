---
name: drupal-token — custom tokens
description: Créer des tokens Drupal custom avec hook_token_info() et hook_tokens() - token types, tokens sur mesure, callback avec entité en contexte.
---

# Custom Tokens — Référence Complète

## hook_token_info() — Déclarer les Types et Tokens

```php
<?php

/**
 * Implements hook_token_info().
 */
function mon_module_token_info(): array {
  $info = [];

  // ── Nouveau type de token ─────────────────────────────────────────────────
  // Utile si vous avez des entités custom (Custom Entity, Webform...)
  $info['types']['commande'] = [
    'name' => t('Commande'),
    'description' => t('Tokens liés aux entités Commande.'),
    'needs-data' => 'commande',    // Clé dans le tableau $data de hook_tokens
  ];

  // ── Tokens pour ce nouveau type ───────────────────────────────────────────
  $info['tokens']['commande']['reference'] = [
    'name' => t('Référence'),
    'description' => t('Numéro de référence de la commande.'),
  ];

  $info['tokens']['commande']['montant'] = [
    'name' => t('Montant'),
    'description' => t('Montant total de la commande.'),
  ];

  $info['tokens']['commande']['client-nom'] = [
    'name' => t('Nom du client'),
    'description' => t('Nom complet du client.'),
  ];

  // ── Tokens ajoutés à un type existant (node) ─────────────────────────────
  $info['tokens']['node']['mot-cles-seo'] = [
    'name' => t('Mots-clés SEO'),
    'description' => t('Mots-clés extraits du contenu pour la méta SEO.'),
    'type' => 'node',
  ];

  // Token avec type retourné (chaînable)
  $info['tokens']['node']['categorie-principale'] = [
    'name' => t('Catégorie principale'),
    'description' => t('Premier terme du champ field_categories.'),
    'type' => 'taxonomy_term',    // Retourne un token de type taxonomy_term → chaînable
  ];

  return $info;
}
```

---

## hook_tokens() — Résoudre les Valeurs

> **⚠️ D8.6+ / D11 — Le 5e paramètre `BubbleableMetadata` est obligatoire.** Toute
> valeur de token dérivée d'une entité DOIT propager ses métadonnées de cache via
> `$bubbleable_metadata->addCacheableDependency($entity)`. Sans ça, le rendu qui
> consomme le token (page, bloc, vue) ne sera **pas invalidé** quand l'entité change
> → contenu périmé servi depuis le cache. C'est le bug n°1 des tokens custom.

```php
<?php

use Drupal\Component\Utility\Html;
use Drupal\Core\Render\BubbleableMetadata;

/**
 * Implements hook_tokens().
 *
 * @param string $type     Type de token (node, commande, user...)
 * @param array  $tokens   Tokens à résoudre : ['reference' => '[commande:reference]', ...]
 * @param array  $data     Contexte : ['commande' => $entity, 'node' => $node, ...]
 * @param array  $options  Options : ['langcode', 'callback', 'clear', 'sanitize']
 * @param \Drupal\Core\Render\BubbleableMetadata $bubbleable_metadata
 *   Collecteur de cache tags/contexts/max-age. OBLIGATOIRE depuis D8.6.
 */
function mon_module_tokens(
  string $type,
  array $tokens,
  array $data,
  array $options,
  BubbleableMetadata $bubbleable_metadata,
): array {
  $replacements = [];
  $sanitize = $options['sanitize'] ?? TRUE;

  $langcode = $options['langcode'] ?? \Drupal::languageManager()->getCurrentLanguage()->getId();
  $token_service = \Drupal::service('token');

  // ── Tokens pour les commandes ─────────────────────────────────────────────
  if ($type === 'commande' && isset($data['commande'])) {
    /** @var \Drupal\mon_module\Entity\Commande $commande */
    $commande = $data['commande'];
    // Propage les cache tags de l'entité : invalidation auto quand elle change.
    $bubbleable_metadata->addCacheableDependency($commande);

    foreach ($tokens as $name => $original) {
      switch ($name) {
        case 'reference':
          $value = $commande->get('reference')->value;
          $replacements[$original] = $sanitize ? Html::escape($value) : $value;
          break;

        case 'montant':
          $montant = (float) $commande->get('montant')->value;
          $replacements[$original] = number_format($montant, 2, ',', ' ') . ' €';
          break;

        case 'client-nom':
          $user = $commande->get('uid')->entity;
          $nom = $user ? $user->getDisplayName() : '';
          // L'affichage dépend aussi de l'utilisateur → propager sa dépendance.
          if ($user) {
            $bubbleable_metadata->addCacheableDependency($user);
          }
          $replacements[$original] = $sanitize ? Html::escape($nom) : $nom;
          break;
      }
    }
  }

  // ── Tokens ajoutés à 'node' ───────────────────────────────────────────────
  if ($type === 'node' && isset($data['node'])) {
    /** @var \Drupal\node\Entity\Node $node */
    $node = $data['node'];
    // Utiliser la traduction courante si disponible
    $node = \Drupal::service('entity.repository')->getTranslationFromContext($node, $langcode);
    $bubbleable_metadata->addCacheableDependency($node);

    foreach ($tokens as $name => $original) {
      switch ($name) {
        case 'mot-cles-seo':
          // Extraire les termes de tags comme mots-clés SEO
          $tags = $node->get('field_tags')->referencedEntities();
          $keywords = array_map(fn($t) => $t->getName(), $tags);
          // Les termes influencent le résultat → propager leur cache.
          foreach ($tags as $tag) {
            $bubbleable_metadata->addCacheableDependency($tag);
          }
          // Token texte simple : neutraliser tout markup résiduel.
          $replacements[$original] = $sanitize
            ? Html::escape(implode(', ', $keywords))
            : implode(', ', $keywords);
          break;

        case 'categorie-principale':
          // Retourner l'entité taxonomy_term pour permettre le chaining
          $field = $node->get('field_categories');
          if (!$field->isEmpty()) {
            $term = $field->entity;
            if ($term) {
              $bubbleable_metadata->addCacheableDependency($term);
              // Trouver les tokens de taxonomy_term chaînés ([…:categorie-principale:name])
              $chained_tokens = $token_service->findWithPrefix($tokens, $name);
              if ($chained_tokens) {
                // generate() reçoit aussi $bubbleable_metadata pour propager le cache
                // des sous-tokens taxonomy_term.
                $replacements += $token_service->generate(
                  'taxonomy_term',
                  $chained_tokens,
                  ['taxonomy_term' => $term],
                  $options,
                  $bubbleable_metadata,
                );
              }
              else {
                $replacements[$original] = $sanitize ? Html::escape($term->getName()) : $term->getName();
              }
            }
          }
          break;
      }
    }
  }

  return $replacements;
}
```

---

## Utiliser les Custom Tokens

```php
// Utilisation dans le code
$commande = \Drupal\mon_module\Entity\Commande::load(42);

$text = 'Commande [commande:reference] — Total : [commande:montant] — Client : [commande:client-nom]';

$result = \Drupal::service('token')->replace(
  $text,
  ['commande' => $commande],
  ['clear' => TRUE]
);
// → 'Commande CMD-2026-001 — Total : 149,99 € — Client : Jean Dupont'

// Dans Webform handler Email
// To: [webform_submission:values:email]
// Subject: Confirmation commande [commande:reference]
// Body: Bonjour [commande:client-nom], votre commande [commande:reference]...

// Token chaînable
$text = 'Catégorie : [node:categorie-principale:name]';
// → 'Catégorie : Drupal'
```

---

## Où déclarer les hooks token

Deux fichiers sont auto-chargés par le système token, sans aucun code de bootstrap :

- **`mon_module.tokens.inc`** — emplacement canonique recommandé (voir `node.tokens.inc`,
  `system.tokens.inc` dans le core). Le module charge ce `.inc` automatiquement avant
  d'invoquer `hook_token_info()` / `hook_tokens()`. Mettre les deux hooks ici garde
  le `.module` léger.
- **`mon_module.module`** — possible aussi, mais à réserver aux très petits modules.

```php
// ❌ Déprécié — ne plus utiliser
module_load_include('inc', 'mon_module', 'mon_module.tokens');

// ✅ Si un chargement manuel est vraiment nécessaire (rarement le cas pour .tokens.inc)
\Drupal::moduleHandler()->loadInclude('mon_module', 'inc', 'mon_module.tokens');
```

> **Ne jamais** déclarer `hook_token_info()` dans `hook_install()` : ce sont des hooks
> de découverte, ils doivent vivre dans `.module` ou `.tokens.inc`, puis `drush cr`.

---

## D11 — Hooks OOP par attribut `#[Hook]`

Drupal 11 permet d'implémenter les hooks token en POO via l'attribut
`#[Hook]` (`Drupal\Core\Hook\Attribute\Hook`), dans une classe sous `src/Hook/`.
Plus de fonctions procédurales, injection de dépendances native, testabilité.
Le procédural reste 100 % supporté en D11 — choisir selon la convention du projet.

```php
<?php

declare(strict_types=1);

namespace Drupal\mon_module\Hook;

use Drupal\Core\Hook\Attribute\Hook;
use Drupal\Core\Render\BubbleableMetadata;
use Drupal\Core\StringTranslation\StringTranslationTrait;

/**
 * Hooks token du module, version OOP (Drupal 11).
 */
final class TokenHooks {

  use StringTranslationTrait;

  #[Hook('token_info')]
  public function tokenInfo(): array {
    return [
      'types' => [
        'commande' => [
          'name' => $this->t('Commande'),
          'description' => $this->t('Tokens liés aux entités Commande.'),
          'needs-data' => 'commande',
        ],
      ],
      'tokens' => [
        'commande' => [
          'reference' => [
            'name' => $this->t('Référence'),
            'description' => $this->t('Numéro de référence de la commande.'),
          ],
        ],
      ],
    ];
  }

  #[Hook('tokens')]
  public function tokens(
    string $type,
    array $tokens,
    array $data,
    array $options,
    BubbleableMetadata $bubbleable_metadata,
  ): array {
    // Même logique que la version procédurale ci-dessus.
    return [];
  }

}
```
