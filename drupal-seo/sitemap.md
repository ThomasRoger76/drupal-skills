---
name: drupal-seo — sitemap XML
description: Configurer drupal/simple_sitemap pour Drupal - sitemaps par entité, image sitemap, sitemaps multilingues, exclusions, et soumission à Google Search Console.
---

# Simple Sitemap — Référence Complète

## Installation

```bash
composer require drupal/simple_sitemap
drush en simple_sitemap -y
```

**Interface :** `/admin/config/search/simplesitemap`

---

## Configuration par Type d'Entité

```yaml
# config/install/simple_sitemap.custom_link.homepage.yml
langcode: fr
status: true
id: homepage
weight: 0
meta:
  path: /
  priority: '1.0'
  changefreq: daily
  include_images: 0
```

```
Interface : /admin/config/search/simplesitemap

Pour chaque type d'entité (node/article, taxonomy_term/tags...) :
  ├── Inclure dans le sitemap : ✅
  ├── Priority : 0.5 (défaut) → 1.0 (homepage) → 0.3 (taxonomy)
  ├── Changefreq : always | hourly | daily | weekly | monthly | yearly | never
  └── Include images : ✅ (pour Google Images)
```

---

## Configuration via YAML

```yaml
# config/install/simple_sitemap.bundle_settings.node.article.yml
langcode: fr
status: true
id: node__article
type: entity
entity_type: node
bundle: article
sitemap_generator: default
priority: '0.8'
changefreq: weekly
include_images: '1'
index: '1'
```

---

## Sitemaps Multilingues

```bash
# Simple Sitemap génère automatiquement des sitemaps par langue
# si le module Language est actif

# Sitemap index : https://mon-site.com/sitemap.xml
# → sitemap_fr.xml, sitemap_en.xml, sitemap_de.xml

# Chaque URL dans le sitemap multilingue inclut des hreflang
```

---

## Générer et Valider

```bash
# Régénérer le sitemap depuis Drush
drush simple-sitemap:generate
drush ssg  # alias

# Vider le sitemap
drush simple-sitemap:remove-sitemap-from-queue
drush simple-sitemap:generate

# Afficher l'URL du sitemap
drush php:eval "echo \Drupal::request()->getSchemeAndHttpHost() . '/sitemap.xml';"

# Vérifier la structure
curl https://mon-site.com/sitemap.xml | head -30

# Valider le sitemap XML
# → https://www.xml-sitemaps.com/validate-xml-sitemap.html

# Soumettre à Google Search Console
# → https://search.google.com/search-console/sitemaps
```

---

## Exclusions

```php
// Exclure des entités spécifiques du sitemap
// Via l'UI : éditer le nœud → "Simple Sitemap" section → décocher

// Via code — hook_simple_sitemap_links_alter() (nom exact, sans suffixe _all)
// $sitemap est l'objet SimpleSitemap (variant) en cours de génération.
function mon_module_simple_sitemap_links_alter(array &$links, $sitemap): void {
  // Exclure les pages de test
  $links = array_filter($links, function ($link) {
    return !str_contains($link['url'], '/test/');
  });
}
```

---

## Image Sitemap

```yaml
# Activer le sitemap d'images (pour Google Images)
# config/install/simple_sitemap.bundle_settings.node.article.yml
include_images: '1'    # Inclut les images du nœud dans le sitemap

# Le sitemap d'images est généré automatiquement si :
# 1. include_images = 1 dans la config
# 2. Le nœud a des champs Image ou Media (image)
# 3. Module "Simple Sitemap: Image" activé
```

---

## Cron — Génération Automatique

```bash
# Simple Sitemap se régénère automatiquement via le cron Drupal
# Configurer la fréquence : /admin/config/search/simplesitemap → Cron

# En production — régénérer après chaque déploiement
drush simple-sitemap:generate && echo "Sitemap régénéré."

# Dans le Makefile de déploiement
deploy:
	composer install --no-dev
	drush deploy
	drush simple-sitemap:generate
```
