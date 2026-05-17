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

```php
<?php

/**
 * Implements hook_tokens().
 *
 * @param string $type     Type de token (node, commande, user...)
 * @param array  $tokens   Tokens à résoudre : ['reference' => '[commande:reference]', ...]
 * @param array  $data     Contexte : ['commande' => $entity, 'node' => $node, ...]
 * @param array  $options  Options : ['langcode', 'callback', 'clear', 'sanitize']
 */
function mon_module_tokens(string $type, array $tokens, array $data, array $options): array {
  $replacements = [];
  $sanitize = $options['sanitize'] ?? TRUE;

  $langcode = $options['langcode'] ?? \Drupal::languageManager()->getCurrentLanguage()->getId();
  $token_service = \Drupal::service('token');

  // ── Tokens pour les commandes ─────────────────────────────────────────────
  if ($type === 'commande' && isset($data['commande'])) {
    /** @var \Drupal\mon_module\Entity\Commande $commande */
    $commande = $data['commande'];

    foreach ($tokens as $name => $original) {
      switch ($name) {
        case 'reference':
          $value = $commande->get('reference')->value;
          $replacements[$original] = $sanitize ? \Drupal\Component\Utility\Html::escape($value) : $value;
          break;

        case 'montant':
          $montant = (float) $commande->get('montant')->value;
          $replacements[$original] = number_format($montant, 2, ',', ' ') . ' €';
          break;

        case 'client-nom':
          $user = $commande->get('uid')->entity;
          $nom = $user ? $user->getDisplayName() : '';
          $replacements[$original] = $sanitize ? \Drupal\Component\Utility\Html::escape($nom) : $nom;
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

    foreach ($tokens as $name => $original) {
      switch ($name) {
        case 'mot-cles-seo':
          // Extraire les termes de tags comme mots-clés SEO
          $tags = $node->get('field_tags')->referencedEntities();
          $keywords = array_map(fn($t) => $t->getName(), $tags);
          $replacements[$original] = implode(', ', $keywords);
          break;

        case 'categorie-principale':
          // Retourner l'entité taxonomy_term pour permettre le chaining
          $field = $node->get('field_categories');
          if (!$field->isEmpty()) {
            $term = $field->entity;
            if ($term) {
              // Trouver les tokens de taxonomy_term qui chainées
              $chained_tokens = $token_service->findWithPrefix($tokens, $name);
              if ($chained_tokens) {
                $replacements += $token_service->generate(
                  'taxonomy_term',
                  $chained_tokens,
                  ['taxonomy_term' => $term],
                  $options
                );
              }
              else {
                $replacements[$original] = $term->getName();
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

## Déclarer les Tokens dans `.module`

```php
// mon_module.module — importer les fonctions token
// hook_token_info et hook_tokens peuvent être dans le .module directement
// OU dans un fichier .tokens.inc

// Pour garder le .module léger :
// mon_module.module
function mon_module_token_info() {
  // Déléguer à un fichier dédié si complexe
  module_load_include('inc', 'mon_module', 'mon_module.tokens');
  return _mon_module_token_info();
}
```
