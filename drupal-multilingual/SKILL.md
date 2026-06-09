---
name: drupal-multilingual
description: Use when setting up multilingual Drupal sites (Language, Content Language, Interface Translation, Configuration Translation modules), configuring language negotiation (URL prefix, domain, session, user preference), translating content programmatically with addTranslation() or getTranslation(), translating configuration (menus, Views, blocks, text formats), building multilingual Views with language filter and translation_language, importing .po translation files with drush locale:import, updating interface translations with drush locale:update, implementing TMGMT for translation workflows, handling RTL languages with CSS logical properties, managing language-specific entity queries with langcode condition, or configuring URL aliases per language in Drupal 8-11+
---

# Drupal Multilingual — Référence Complète

## Overview

Référentiel complet du multilinguisme Drupal 8-11+ : pile de modules multilinguisme, négociation de langue, traduction de contenu, traduction de config, Views multilingues, TMGMT, propriétés logiques CSS pour RTL, API programmatique. Drupal a le système de traduction le plus complet de tous les CMS.

## 🏗️ La Pile de Modules Multilinguisme

```
┌─────────────────────────────────────────────────────┐
│  Language (core) — gestion des langues du site      │
├─────────────────────────────────────────────────────┤
│  Content Language (core) — traduit les entités      │
├─────────────────────────────────────────────────────┤
│  Interface Translation (core) — traduit l'UI PHP    │
├─────────────────────────────────────────────────────┤
│  Configuration Translation (core) — traduit config  │
└─────────────────────────────────────────────────────┘
       ↓ optionnel
┌─────────────────────────────────────────────────────┐
│  TMGMT (contrib) — workflow de traduction           │
│  Language Switcher Block — bloc natif               │
│  Pathauto — alias URL par langue                    │
└─────────────────────────────────────────────────────┘
```

**Règle :** activer les 4 modules core dans cet ordre. Configurer Language en premier.

> **Convention d'exécution — Docker natif.** Toutes les commandes `drush` de ce skill
> se lancent via `docker compose exec php drush ...` (jamais `ddev`). Adapter le nom du
> service (`php`, `app`...) au `docker-compose.yml` du projet.

---

## Quick Decision Table

| Besoin | Outil | Référence |
|--------|-------|-----------|
| Ajouter une langue au site | `/admin/config/regional/language` + Language module | [multilingual-setup.md](multilingual-setup.md) |
| URL avec préfixe langue (/fr/, /en/) | Language Negotiation → URL (prefix) | [language-negotiation.md](language-negotiation.md) |
| URL avec domaine différent par langue | Language Negotiation → Domain | [language-negotiation.md](language-negotiation.md) |
| Détecter la langue depuis le navigateur | Language Negotiation → Browser | [language-negotiation.md](language-negotiation.md) |
| Rendre un content type traduisible | Content Language → configurer les champs traduisibles | [content-translation.md](content-translation.md) |
| Créer une traduction programmatiquement | `$node->addTranslation('fr', [...])` | [content-translation.md](content-translation.md) |
| Charger la traduction courante d'un nœud | `$node->getTranslation($langcode)` | [content-translation.md](content-translation.md) |
| Charger la traduction du contexte courant | `entity.repository->getTranslationFromContext($entity)` | [content-translation.md](content-translation.md) |
| Vérifier si une traduction existe | `$node->hasTranslation('fr')` | [content-translation.md](content-translation.md) |
| EntityQuery filtré par langue | `->condition('langcode', 'fr')` | [content-translation.md](content-translation.md) |
| Lister les traductions d'un nœud | `$node->getTranslationLanguages()` | [content-translation.md](content-translation.md) |
| Traduire un menu, un bloc, une View | Configuration Translation module + UI | [config-translation.md](config-translation.md) |
| Traduire la config programmatiquement | `LanguageManager::getLanguageConfigOverride()` | [config-translation.md](config-translation.md) |
| Traduire les chaînes d'un module | Interface Translation module + fichiers `.po` | [interface-translation.md](interface-translation.md) |
| Importer des traductions `.po` | `docker compose exec php drush locale:import fr fichier.po` | [interface-translation.md](interface-translation.md) |
| Mettre à jour les traductions contrib | `docker compose exec php drush locale:update` | [interface-translation.md](interface-translation.md) |
| Traduire une chaîne dans PHP | `$this->t('Chaîne', [], ['langcode' => 'fr'])` | [interface-translation.md](interface-translation.md) |
| Views filtrées par langue courante | Field Language → "Interface text language selected for page" | [multilingual-views.md](multilingual-views.md) |
| Views avec sélecteur de langue exposé | Exposed filter sur `langcode` | [multilingual-views.md](multilingual-views.md) |
| Views montrant la traduction ou l'original | Translation Language → "Current user's language" | [multilingual-views.md](multilingual-views.md) |
| Workflow de traduction avec prestataire | TMGMT module + connecteur (DeepL, Gengo...) | [tmgmt.md](tmgmt.md) |
| Envoyer du contenu à traduire | TMGMT → Job → Translator | [tmgmt.md](tmgmt.md) |
| CSS adapté RTL automatiquement | CSS Logical Properties (margin-inline-start...) | [multilingual-setup.md](multilingual-setup.md) |
| Alias URL par langue (/fr/mon-article) | Pathauto + Language module | [multilingual-setup.md](multilingual-setup.md) |
| Générer une URL dans une langue spécifique | `Url::fromRoute(..., [], ['language' => $language])` | [content-translation.md](content-translation.md) |
| Langue fallback si traduction manquante | Language Fallback dans Content Language config | [multilingual-setup.md](multilingual-setup.md) |
| **`drush locale:import` — màj sans écraser les traductions custom** | `--override=none` (1er import) vs `--override=all` (mise à jour — remplace tout) | [interface-translation.md](interface-translation.md) |
| **Détecter la langue depuis l'IP (geolocation)** | Language Negotiation → module contrib `drupal/smart_ip` + `drupal/geoip` (résolution IP→pays puis map pays→langue) | [language-negotiation.md](language-negotiation.md) |
| **Tester la langue courante dans un préprocess** | `$language = \Drupal::service('language_manager')->getCurrentLanguage()->getId()` | [content-translation.md](content-translation.md) |
| **Cache multilingual — contexte manquant** | Toujours `#cache['contexts'][] = 'languages:language_interface'` sur les render arrays avec texte traduit | [multilingual-setup.md](multilingual-setup.md) |
| **Importer du contenu traduit en masse** | Migrate API — `destination.translations: true` dans le YAML migrate | [content-translation.md](content-translation.md) |
| **Views — n'afficher que le contenu traduit dans la langue courante** | Views → Filters → "Translation Language" → "Current user's language" (pas l'original) | [multilingual-views.md](multilingual-views.md) |
| **Language Switcher bloc avec flag images** | Module `drupal/languageicons` ou CSS background-image sur `.language-link` | [multilingual-setup.md](multilingual-setup.md) |

## Anti-Patterns Critiques

| ❌ À ne jamais faire | ✅ Bonne pratique | Impact |
|---------------------|------------------|--------|
| Stocker `langcode` dans une variable PHP globale | `\Drupal::languageManager()->getCurrentLanguage()` | Langue incorrecte en contexte CLI/Cron |
| `$node->field_title->value` sans `getTranslation()` | `$node->getTranslation($langcode)->field_title->value` | Valeur en langue d'origine toujours |
| Views sans configuration de langue | Toujours configurer "Translation Language" dans Views | Même contenu affiché quelle que soit la langue |
| `hook_preprocess_node` sans contexte langue | `entity.repository->getTranslationFromContext($node)` | Champs en langue d'origine |
| Oublier `#cache['contexts'][] = 'languages:language_interface'` | Toujours ajouter ce contexte sur les render arrays multilingues | Mauvaise langue en cache |
| Config Translation activé sans Content Language | Activer Content Language en premier | Config Translation dépend du contenu traduisible |
| `drush locale:import` sans `--override=all` si mise à jour | Utiliser `--override=all` pour les mises à jour | Anciennes traductions non remplacées |
| Text format "Full HTML" non traduit | Traduire les labels des text formats | Éditeurs voient l'interface en langue d'origine |
| Alias URL sans langue prefix | Configurer Pathauto par langue | URLs sans contexte langue = SEO dupliqué |
| Champ `langcode` non inclus dans les migrations | `langcode: fr` explicite dans les YAMLs migrate | Contenu sans langue, non filtrable |

## Évolution par Version Majeure

| Feature | D8 | D9 | D10 | D11 |
|---------|----|----|-----|-----|
| 4 modules core multilinguisme | ✅ | ✅ | ✅ | ✅ |
| Language Negotiation | ✅ | ✅ | ✅ | ✅ |
| Config Translation | ✅ | ✅ | ✅ | ✅ |
| `getTranslationFromContext()` | ✅ | ✅ | ✅ | ✅ |
| TMGMT (contrib) | ✅ | ✅ | ✅ | ✅ |
| Language fallback | ✅ | ✅ | ✅ | ✅ |
| Views Translation Language | ✅ | ✅ | ✅ | ✅ |
| JSON:API pour contenu traduit | contrib | contrib | ✅ amélioré | ✅ |
| `drush locale:update` | ✅ | ✅ | ✅ | ✅ |

## Auto-Amélioration

- **[lessons.md](lessons.md)** — Bugs multilingues découverts en projet réel.
- **[CHANGELOG.md](CHANGELOG.md)** — Historique des versions.

## See Also

- `drupal-core` — Entity API, `getTranslation()`, `StringTranslationTrait`
- `drupal-theming` — CSS Logical Properties pour RTL, `is_front` multilingue
- `drupal-views` — Views multilingues, Translation Language, exposed langcode filter
- `drupal-config` — Config Translation, `language_manager->getLanguageConfigOverride()`
- `drupal-migration` — Migrations multilingues, `langcode` dans YAML, `translations: true`
- `drupal-api` — JSON:API avec contenu traduit, `Accept-Language` header
