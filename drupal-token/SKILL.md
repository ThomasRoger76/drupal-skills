---
name: drupal-token
description: Use when working with Drupal Token module - using built-in tokens in Metatag ([node:title], [site:name]), Pathauto ([node:field_category:entity:name]), Webform ([webform_submission:values:email]), or email templates, creating custom token types and tokens with hook_token_info() and hook_tokens(), implementing token replacement with token.replace() service, using tokens in configuration YAML, using the Token Tree browser UI to discover available tokens, debugging token output with [token:debug], or creating tokens that expose entity field values in Drupal 8-11+
---

# Drupal Token — Référence Complète

## Overview

Référentiel complet du module Token Drupal 8-11+ : tokens natifs (node, user, site, date), utilisation dans Metatag/Pathauto/Webform/Email, création de tokens custom (`hook_token_info()`, `hook_tokens()`), remplacement programmatique. Token est la colle entre les modules de configuration.

> **Exemples PHP complets** (`hook_token_info()` + `hook_tokens()` avec chaining vers une entité) dans [custom-tokens.md](custom-tokens.md).

## 🎯 La Règle Fondamentale

> **Token = variable universelle Drupal.** Partout où une configuration accepte du texte dynamique (Metatag, Pathauto, Webform, Email, Rules...), les tokens permettent d'injecter des valeurs d'entités sans écrire de code. Créer des tokens custom quand les tokens natifs ne suffisent pas.

---

## Quick Decision Table

| Besoin | Token | Référence |
|--------|-------|-----------|
| Titre de la page courante | `[current-page:title]` | [token-basics.md](token-basics.md) |
| URL du nœud courant | `[node:url]` | [token-basics.md](token-basics.md) |
| Résumé du body du nœud | `[node:body:summary]` | [token-basics.md](token-basics.md) |
| Nom du site | `[site:name]` | [token-basics.md](token-basics.md) |
| Nom de l'auteur du nœud | `[node:author:name]` | [token-basics.md](token-basics.md) |
| Date de création (format ISO) | `[node:created:html_datetime]` | [token-basics.md](token-basics.md) |
| Valeur d'un champ custom | `[node:field_mon_champ]` | [token-basics.md](token-basics.md) |
| Terme d'un champ Entity Reference | `[node:field_tags:entity:name]` | [token-basics.md](token-basics.md) |
| URL de l'image du nœud | `[node:field_image:entity:file:url]` | [token-basics.md](token-basics.md) |
| Email de l'utilisateur | `[current-user:mail]` | [token-basics.md](token-basics.md) |
| Valeur d'une soumission Webform | `[webform_submission:values:email]` | [token-basics.md](token-basics.md) |
| Découvrir les tokens disponibles | Token Tree UI → `/admin/help/token` | [token-basics.md](token-basics.md) |
| Créer un token custom (type + token) | `hook_token_info()` + `hook_tokens()` | [custom-tokens.md](custom-tokens.md) |
| Remplacer les tokens dans une string PHP | `\Drupal::service('token')->replace()` | [custom-tokens.md](custom-tokens.md) |
| Token avec un entité en contexte | `token->replace($text, ['node' => $node])` | [custom-tokens.md](custom-tokens.md) |
| Nettoyer les tokens non résolus | `token->replace($text, [], ['clear' => TRUE])` | [custom-tokens.md](custom-tokens.md) |
| Tokens dans Pathauto | `[node:title]` + `[node:field_categorie:entity:name]` | [tokens-in-modules.md](tokens-in-modules.md) |
| Tokens dans Metatag | `[node:title]` + `[node:summary]` + `[node:field_image:entity:file:url]` | [tokens-in-modules.md](tokens-in-modules.md) |
| Tokens dans les emails Webform | `[webform_submission:values:prenom]` | [tokens-in-modules.md](tokens-in-modules.md) |
| Tester un token depuis Drush | `drush php:eval "echo \Drupal::token()->replace('[node:title]', ['node' => \Drupal::entityTypeManager()->getStorage('node')->load(42)]);"` | [token-basics.md](token-basics.md) |
| **Déboguer un token non résolu** | Vérifier `hook_token_info()` retourne bien le type, `drush cr` | [custom-tokens.md](custom-tokens.md) |
| **Token custom sur entité custom** | `hook_token_info()` + `hook_tokens()` PHP complet avec `$data['mon_type']` | [custom-tokens.md](custom-tokens.md) |
| **Token chaînable vers une entité** | `'type' => 'taxonomy_term'` + `$token_service->findWithPrefix($tokens, $name)` | [custom-tokens.md](custom-tokens.md) |
| **Cacher le résultat d'un token lent** | `cache.default` dans `hook_tokens()` avec cache tag de l'entité | [custom-tokens.md](custom-tokens.md) |
| Lister tous les tokens disponibles (UI) | `/admin/help/token` — Token Tree navigable par type | [token-basics.md](token-basics.md) |
| Token avec fallback si vide | `[node:summary\|[node:title]]` (pipe = fallback) | [token-basics.md](token-basics.md) |

## Anti-Patterns Critiques

| ❌ À ne jamais faire | ✅ Bonne pratique | Impact |
|---------------------|------------------|--------|
| Hardcoder le nom du site dans Metatag | `[site:name]` via token | Doit être mis à jour manuellement après changement |
| `[node:title]` sans fallback | `[node:title\|[site:name]]` (token chain) | Titre vide si nœud sans titre |
| Tokens sans `['clear' => TRUE]` si optionnel | Toujours `clear: true` pour les tokens qui peuvent être vides | `[node:field_optionnel]` laissé tel quel dans le texte |
| Créer un token avec un callback lent | Cacher le résultat ou utiliser `#lazy_builder` | Ralentit chaque remplacement de token |
| Token type `id` qui clash avec un type existant | Préfixer avec le nom du module : `mon_module_commande` | Conflits avec d'autres modules |
| Utiliser `token_replace()` (fonction D7) | `\Drupal::token()->replace()` (service D8+) | Fonction dépréciée/supprimée |

## Évolution par Version Majeure

| Feature | D8 | D9 | D10 | D11 |
|---------|----|----|-----|-----|
| Module Token (contrib) | ✅ | ✅ | ✅ | ✅ |
| Token Tree UI | ✅ | ✅ | ✅ | ✅ |
| `hook_token_info()` | ✅ | ✅ | ✅ | ✅ |
| `hook_tokens()` | ✅ | ✅ | ✅ | ✅ |
| Tokens natifs (core) | ✅ limité | ✅ | ✅ | ✅ |
| Token chaining (`\|`) | ✅ | ✅ | ✅ | ✅ |
| `token_replace()` (D7 compat) | ✅ | ⚠️ | ❌ | ❌ |

## Auto-Amélioration

- **[lessons.md](lessons.md)** — Tokens mal configurés découverts en projet.
- **[CHANGELOG.md](CHANGELOG.md)** — Historique des versions.

## See Also

- `drupal-seo` — Tokens dans Metatag, Pathauto patterns
- `drupal-webform` — Tokens dans les handlers Email et Remote POST
- `drupal-core` — `hook_token_info()`, `hook_tokens()`, service token
- `drupal-multilingual` — Tokens dans les contextes multilingues
