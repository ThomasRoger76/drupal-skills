---
name: drupal-multilingual — setup
description: Configuration complète de la pile multilingue Drupal (4 modules core), ajout de langues, négociation de langue, URL prefixes, language fallback, et CSS logical properties pour RTL.
---

# Setup Multilingue Drupal — Guide Complet

## Les 4 Modules Core à Activer (Ordre Obligatoire)

```bash
# 1. Language — gestion des langues
# 2. Content Language — traduction des entités
# 3. Interface Translation — traduction de l'UI
# 4. Configuration Translation — traduction de la config

drush en language content_translation locale config_translation -y

# Vérifier l'activation
drush pm:list --status=enabled | grep -E "language|locale|config_translation|content_translation"
```

**Pourquoi cet ordre :** `content_translation` et `config_translation` dépendent de `language`. `locale` (Interface Translation) est indépendant mais doit être activé avant `config_translation`.

---

## Ajouter des Langues

```bash
# Via Drush
drush language:add fr
drush language:add en
drush language:add de

# Définir la langue par défaut du site
drush config:set system.site langcode fr -y
drush config:set system.site default_langcode fr -y
```

```yaml
# config/install/language.entity.language.fr.yml
langcode: fr
status: true
id: fr
label: French
direction: ltr    # 'ltr' ou 'rtl' pour les langues RTL (arabe, hébreu...)
weight: 0
locked: false
```

---

## Configuration de la Négociation de Langue

```yaml
# config/install/language.negotiation.yml
langcode: fr
status: true
id: language_interface

# Méthodes de détection, par ordre de priorité :
method_weights_language_interface:
  url: -8              # Priorité la plus haute : URL préfixe ou domaine
  session: -6
  user: -4
  browser: -2          # Détection depuis Accept-Language header
  selected_langcode: 12
```

**Via l'UI :** `/admin/config/regional/language/detection`

### URL Prefix — Configuration

```yaml
# config/install/language.negotiation.yml
method_weights_language_url:
  prefixes:
    fr: fr         # /fr/chemin → langue française
    en: en         # /en/path → langue anglaise
    de: de         # /de/pfad → langue allemande
  domains: {}      # Pas de domaines séparés
```

### Domain — Configuration (Sites Multidomaines)

```yaml
method_weights_language_url:
  prefixes: {}     # Pas de préfixes
  domains:
    fr: fr.mon-site.com
    en: en.mon-site.com
    de: de.mon-site.com
```

**Note :** le mode domaine nécessite des DNS et certificats SSL par domaine. Le mode préfixe est plus simple.

---

## Language Fallback

```yaml
# config/install/language.fallback.yml
langcode: fr
status: true
id: language_content
fallbacks:
  fr:              # Si FR manque, chercher dans cet ordre :
    - fr
    - en           # Fallback vers l'anglais
    - und          # Puis vers "langue non définie"
  de:
    - de
    - fr
    - en
  en:
    - en
    - und
```

**Via l'UI :** `/admin/config/regional/content-language` → "Add language" → configurer le fallback.

---

## Langues RTL — Setup CSS

Pour les langues RTL (arabe `ar`, hébreu `he`, persan `fa`, ourdou `ur`) :

```php
// Détecter la direction dans le preprocess
function mon_theme_preprocess_html(array &$variables): void {
  $language = \Drupal::languageManager()->getCurrentLanguage();
  $variables['html_attributes']['lang'] = $language->getId();
  $variables['html_attributes']['dir'] = $language->getDirection(); // 'ltr' ou 'rtl'

  // Cache context obligatoire
  $variables['#cache']['contexts'][] = 'languages:language_interface';
}
```

```css
/* ✅ CSS Logical Properties — adapté RTL/LTR automatiquement */
.card {
  /* Remplace margin-left/right selon dir="rtl" ou dir="ltr" */
  margin-inline-start: 16px;    /* = margin-left en LTR, margin-right en RTL */
  margin-inline-end: 8px;
  padding-inline: 24px 16px;   /* padding-left / padding-right */

  /* Remplace border-left/right */
  border-inline-start: 3px solid var(--color-primary);

  /* Remplace left/right dans les positions */
  inset-inline-start: 0;
  inset-inline-end: auto;

  /* Remplace text-align: left */
  text-align: start;
}

/* ❌ Physique — NE PAS UTILISER sur un site multilingue RTL */
.card {
  margin-left: 16px;
  border-left: 3px solid blue;
  text-align: left;
}

/* ✅ Pour les cas où RTL nécessite une surcharge spécifique */
[dir="rtl"] .icon-arrow {
  transform: scaleX(-1);   /* Retourner les flèches directionnelles */
}
```

---

## Pathauto — Alias URL par Langue

```bash
# Installer Pathauto pour les alias URL
composer require drupal/pathauto
drush en pathauto -y
```

```yaml
# config/install/pathauto.pattern.node_article_fr.yml
langcode: fr
status: true
id: node_article_fr
label: 'Articles FR'
type: 'canonical_entities:node'
pattern: '[node:type]/[node:title]'
selection_criteria:
  bundle:
    id: condition.bundle
    negate: false
    bundles:
      article: article
  language:
    id: condition.language
    negate: false
    langcodes:
      fr: fr
```

**Résultat :** `/article/mon-titre-en-francais` pour les nœuds en français, `/article/my-title-in-english` pour l'anglais.

---

## Bloc de Basculement de Langue

```yaml
# Placer le Language Switcher block dans une région
# config/install/block.block.language_switcher.yml
langcode: fr
status: true
id: language_switcher
theme: mon_theme
region: header
plugin: language_block:language_interface
settings:
  label: 'Basculer la langue'
  label_display: '0'
visibility: {}
```

---

## Commandes Drush Utiles

```bash
# Lister les langues
drush language:info

# Ajouter une langue
drush language:add es

# Supprimer une langue (avec tous ses contenus traduits !)
drush language:delete es

# Importer des traductions .po
drush locale:import fr fichier.fr.po

# Mettre à jour les traductions contrib depuis drupal.org
drush locale:update

# Exporter les traductions personnalisées
drush locale:export fr --types=customized > custom.fr.po

# Vérifier le statut des traductions
drush locale:check
```

---

## Points de Vérification Post-Setup

```bash
# 1. Les 4 modules sont actifs
drush pm:list --status=enabled | grep -E "language|locale|config_translation|content_translation"

# 2. La langue par défaut est correcte
drush config:get system.site langcode

# 3. La négociation de langue est configurée
drush php:eval "var_dump(\Drupal::service('language.negotiator')->getNegotiationMethods());"

# 4. Le préfixe URL fonctionne
curl -I https://mon-site.com/fr/ | grep -i "content-language\|x-drupal"

# 5. Les traductions de config sont actives
drush php:eval "echo \Drupal::moduleHandler()->moduleExists('config_translation') ? 'OK' : 'MANQUE';"

# 6. Watchdog pour les erreurs de langue
drush watchdog:show --type=locale --count=20
```
