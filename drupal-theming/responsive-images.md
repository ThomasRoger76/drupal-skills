# Images Responsives & Breakpoints

## `breakpoints.yml` — Définir les Points de Rupture

```yaml
# mon_theme.breakpoints.yml
# Nommage : THEME.NOM_BREAKPOINT
# Deux approches valides : mobile-first (min-width seul) OU ranges (min+max)

# ── Approche 1 : Mobile-first (recommandée pour les nouveaux projets) ──────────
mon_theme.mobile:
  label: 'Mobile'
  mediaQuery: 'screen and (min-width: 0px)'
  weight: 0
  multipliers:
    - 1x
    - 2x               # Pour les écrans Retina/HiDPI

mon_theme.tablet:
  label: 'Tablette'
  mediaQuery: 'screen and (min-width: 768px)'
  weight: 1
  multipliers:
    - 1x
    - 2x

# ── Approche 2 : Ranges (projets existants, Bootstrap-based) ──────────────────
# (pattern réel des projets Bootstrap5 comme ccilemans)
mon_theme.extra_small:
  label: 'Mobile XS'
  mediaQuery: ''          # Pas de media query = toutes tailles
  weight: 0
  multipliers:
    - 1x

mon_theme.small:
  label: 'Mobile SM'
  mediaQuery: 'all and (min-width: 576px) and (max-width: 767px)'
  weight: 1
  multipliers:
    - 1x

mon_theme.medium:
  label: 'Tablette'
  mediaQuery: 'all and (min-width: 768px) and (max-width: 991px)'
  weight: 2
  multipliers:
    - 1x

mon_theme.desktop:
  label: 'Bureau'
  mediaQuery: 'screen and (min-width: 1024px)'
  weight: 2
  multipliers:
    - 1x
    - 2x

mon_theme.wide:
  label: 'Grand écran'
  mediaQuery: 'screen and (min-width: 1440px)'
  weight: 3
  multipliers:
    - 1x
```

**Important :** après modification du `.breakpoints.yml`, vider le cache ET aller dans  
`/admin/config/media/responsive-image` pour reconfigurer les Responsive Image Styles.

---

## Pipeline Complet — Images Responsives Drupal

### Étape 1 : Image Styles (`/admin/config/media/image-styles`)

Créer un style par taille nécessaire :

| Nom | Effet | Dimensions |
|-----|-------|-----------|
| `article_mobile` | Scale and crop | 400×300 |
| `article_tablet` | Scale and crop | 768×500 |
| `article_desktop` | Scale and crop | 1200×600 |
| `article_wide` | Scale and crop | 1600×800 |
| `thumbnail_square` | Scale and crop | 200×200 |

**D10+ : WebP automatique**  
Dans `/admin/config/media/image-styles`, activer "Convert to WebP" comme effet supplémentaire — Drupal génère les deux versions (original + WebP) avec `<picture>` automatique.

### Étape 2 : Responsive Image Style (`/admin/config/media/responsive-image`)

Créer un Responsive Image Style qui mappe les breakpoints → Image Styles :

```
Nom : Article Hero
Breakpoint group : mon_theme

Mappings :
  mon_theme.wide    → 1x: article_wide
  mon_theme.desktop → 1x: article_desktop
  mon_theme.tablet  → 1x: article_tablet
  mon_theme.mobile  → 1x: article_mobile  (Fallback)

Image de fallback : article_mobile
```

### Étape 3 : Configurer le Formatter dans "Manage Display"

Sur le Content Type (`/admin/structure/types/manage/article/display`) :
- Champ image → Format : **Responsive image**
- Choisir le Responsive Image Style créé

Drupal génère automatiquement :
```html
<picture>
  <source srcset="/files/styles/article_wide/..." media="(min-width: 1440px)" type="image/webp">
  <source srcset="/files/styles/article_desktop/..." media="(min-width: 1024px)" type="image/webp">
  <source srcset="/files/styles/article_tablet/..." media="(min-width: 768px)" type="image/webp">
  <img src="/files/styles/article_mobile/..." loading="lazy" alt="..." width="400" height="300">
</picture>
```

---

## Accéder aux Images dans Twig

```twig
{# Via content (formatter appliqué — recommandé) #}
{{ content.field_image }}

{# Afficher l'image responsive dans un contexte custom #}
{% set image = node.field_image %}
{% if image.entity %}
  {{
    drupal_image(
      image.entity.uri.value,
      'article_desktop',
      {
        alt: image.alt|default(node.label),
        title: image.title
      }
    )
  }}
{% endif %}

{# Accès direct aux propriétés de l'image #}
{{ node.field_image.entity.uri.value }}    {# URI : public://mon-image.jpg #}
{{ node.field_image.alt }}                  {# Texte alternatif #}
{{ node.field_image.title }}                {# Titre #}
{{ node.field_image.width }}                {# Largeur originale #}
{{ node.field_image.height }}               {# Hauteur originale #}

{# URL d'un style d'image spécifique depuis PHP (dans preprocess) #}
```

```php
// Dans preprocess_node — générer l'URL d'un style d'image
function mon_theme_preprocess_node(array &$variables): void {
  $node = $variables['node'];
  if (!$node->field_image->isEmpty()) {
    $image = $node->field_image->entity;
    $style = \Drupal::entityTypeManager()
      ->getStorage('image_style')
      ->load('article_thumbnail');

    if ($style && $image) {
      $variables['thumbnail_url'] = $style->buildUrl($image->uri->value);
    }
  }
}
```

---

## Lazy Loading — HTML Natif

En D10+, Drupal ajoute `loading="lazy"` automatiquement aux images dans les Responsive Image Styles. Pour les images en `.theme` ou dans du code custom :

```php
// Dans preprocess
$variables['#attached']['library'][] = 'mon_theme/lazy-images';
```

```twig
{# Dans un template — loading="lazy" natif #}
<img
  src="{{ file_url(node.field_image.entity.uri.value) }}"
  alt="{{ node.field_image.alt }}"
  loading="lazy"
  width="{{ node.field_image.width }}"
  height="{{ node.field_image.height }}"
>
```

**Toujours préciser `width` et `height`** pour éviter le Cumulative Layout Shift (CLS).

---

## Images de Fond CSS — Pattern Thème

Pour les images de fond définies en CSS, utiliser les CSS custom properties ou des classes de contexte depuis le preprocess :

```php
// Preprocess
function mon_theme_preprocess_node(array &$variables): void {
  $node = $variables['node'];
  if (!$node->field_background->isEmpty()) {
    $image     = $node->field_background->entity;
    $style     = \Drupal::entityTypeManager()->getStorage('image_style')->load('hero_background');
    // file_create_url() supprimé D10 — utiliser file_url_generator
    $image_url = $style
      ? $style->buildUrl($image->uri->value)
      : \Drupal::service('file_url_generator')->generateAbsoluteString($image->uri->value);

    $variables['hero_image_url'] = $image_url;
  }
}
```

```twig
{# Template #}
{% if hero_image_url %}
  <div class="hero" style="--hero-bg: url('{{ hero_image_url }}')">
    {{ content|without('field_background') }}
  </div>
{% endif %}
```

```css
/* CSS */
.hero {
  background-image: var(--hero-bg);
  background-size: cover;
  background-position: center;
}
```

---

## Focal Point (module contrib)

Le module `focal_point` permet à l'éditeur de définir le point de focus d'une image.
Le crop est appliqué automatiquement lors de la génération de l'Image Style.

```bash
composer require drupal/focal_point
docker compose exec php drush en focal_point -y
```

Dans l'Image Style : ajouter l'effet **"Focal Point Scale and Crop"** à la place de "Scale and Crop" standard.

Dans le widget de champ image : activer "Focal Point" — une croix repositionnable apparaît sur l'aperçu.

---

## Breakpoint Groups et le Module Responsive Image

Le module `responsive_image` (core) utilise les groupes de breakpoints définis dans `.breakpoints.yml`.

```bash
docker compose exec php drush en responsive_image -y   # Généralement déjà activé sur les sites front-end
```

Vérifier que le groupe `mon_theme` est disponible dans `/admin/config/media/responsive-image`.

Si les breakpoints ne s'affichent pas après modification du `.breakpoints.yml` :
```bash
docker compose exec php drush cr   # Vider le cache — les breakpoints sont mis en cache
```
