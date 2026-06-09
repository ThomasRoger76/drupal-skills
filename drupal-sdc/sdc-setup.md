---
name: drupal-sdc — setup
description: Créer un Single Directory Component Drupal - structure *.component.yml, props typées, slots, CSS/JS scopés, et activation en dev.
---

# SDC Setup — Créer un Composant

## Prérequis

```bash
# SDC est dans le core Drupal depuis 10.1 (expérimental) et stable D10.3+
# Pas de module à installer — activer dans le thème

# Dans le thème : créer le répertoire components/
mkdir -p web/themes/custom/mon_theme/components
```

---

## Structure Complète d'un Composant

```
web/themes/custom/mon_theme/components/
└── card/
    ├── card.component.yml     ← Définition obligatoire
    ├── card.twig              ← Template HTML (même nom que le répertoire)
    ├── card.css               ← Styles (automatiquement attachés)
    └── card.js                ← JS (automatiquement attaché)
```

---

## `card.component.yml` — Définition du Composant

```yaml
# card.component.yml
$schema: 'https://git.drupalcode.org/project/drupal/-/raw/HEAD/core/assets/schemas/v1/metadata.schema.json'

name: Card
description: 'Composant carte pour afficher un article ou un produit.'
group: Composants

# Props — données passées au composant (typées, validées)
props:
  type: object
  required:
    - title
    - url
  properties:
    title:
      type: string
      title: 'Titre'
      description: 'Titre principal de la carte.'
    url:
      type: string
      title: 'URL'
      description: 'URL de destination.'
      format: uri
    summary:
      type: string
      title: 'Résumé'
      description: 'Texte de description (optionnel).'
    date:
      type: string
      title: 'Date'
      description: 'Date de publication (format texte).'
    image_url:
      type: string
      title: 'URL image'
      format: uri
    image_alt:
      type: string
      title: 'Alt image'
    variant:
      type: string
      title: 'Variante'
      enum: ['default', 'featured', 'compact']
      default: 'default'

# Slots — régions Twig injectables (comme des render arrays Drupal)
slots:
  badge:
    title: 'Badge'
    description: 'Badge optionnel (ex: "Nouveau", catégorie...).'
  footer:
    title: 'Pied de carte'
    description: 'Actions ou informations complémentaires.'

# Librairies — card.css / card.js sont attachés automatiquement.
# Pour dépendre d'une librairie partagée du thème (ex: tokens design) :
libraryOverrides:
  dependencies:
    - mon_theme/design-tokens
```

---

## `card.twig` — Template du Composant

```twig
{#
  card.twig
  Props disponibles : title, url, summary, date, image_url, image_alt, variant
  Slots disponibles : badge, footer
#}

{%
  set classes = [
    'card',
    'card--' ~ (variant|default('default'))|clean_class,
  ]
%}

<article{{ attributes.addClass(classes) }}>
  {% if image_url %}
    <div class="card__image">
      <a href="{{ url }}">
        <img
          src="{{ image_url }}"
          alt="{{ image_alt|default('') }}"
          loading="lazy"
          width="400"
          height="250"
        >
      </a>
    </div>
  {% endif %}

  <div class="card__content">
    {% if badge %}
      <div class="card__badge">
        {{ badge }}
      </div>
    {% endif %}

    <h3 class="card__title">
      <a href="{{ url }}" class="card__link">{{ title }}</a>
    </h3>

    {% if date %}
      <time class="card__date">{{ date }}</time>
    {% endif %}

    {% if summary %}
      <p class="card__summary">{{ summary }}</p>
    {% endif %}

    {% if footer %}
      <footer class="card__footer">
        {{ footer }}
      </footer>
    {% endif %}
  </div>
</article>
```

---

## `card.css` — Styles Scopés

```css
/* card.css — automatiquement chargé quand le composant est utilisé */
.card {
  display: flex;
  flex-direction: column;
  border: 1px solid var(--color-border, #e0e0e0);
  border-radius: 8px;
  overflow: hidden;
  background: var(--color-surface, #fff);
  transition: box-shadow 0.2s;
}

.card:hover {
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.12);
}

.card__image img {
  width: 100%;
  height: 200px;
  object-fit: cover;
  display: block;
}

.card__content {
  padding: 1.25rem;
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.card__title {
  font-size: 1.125rem;
  font-weight: 600;
  margin: 0;
}

.card__link {
  color: inherit;
  text-decoration: none;
}

.card__link:hover {
  color: var(--color-primary);
}

.card__summary {
  color: var(--color-text-muted, #666);
  font-size: 0.9rem;
  line-height: 1.5;
  margin: 0;
}

.card__footer {
  margin-top: auto;
  padding-top: 0.75rem;
  border-top: 1px solid var(--color-border);
}

/* Variante featured */
.card--featured {
  border-color: var(--color-primary);
  border-width: 2px;
}
```

---

## `card.js` — Comportement JS

```javascript
// card.js — automatiquement chargé avec le composant
(function (Drupal, once) {
  'use strict';

  Drupal.behaviors.cardComponent = {
    attach(context) {
      once('card-hover', '.card', context).forEach((card) => {
        card.addEventListener('mouseenter', () => {
          card.classList.add('card--hovered');
        });
        card.addEventListener('mouseleave', () => {
          card.classList.remove('card--hovered');
        });
      });
    },
  };
})(Drupal, once);
```

---

## Activer le Debug SDC (Validation Stricte)

```yaml
# web/sites/default/services.yml (dev)
parameters:
  # Valide les props/slots contre le schema JSON du *.component.yml.
  # Lève une exception explicite si un type ne correspond pas.
  sdc.enforce_schemas: true
```

> **Nom du paramètre :** c'est bien `sdc.enforce_schemas` (core D10.3+), PAS `sdc.debug` qui n'existe pas.
> En production on le laisse à `false` (défaut) pour ne pas planter le rendu sur une prop invalide.

```bash
# Lister tous les composants SDC disponibles
# Service réel : plugin.manager.sdc (classe ComponentPluginManager)
drush php:eval "
\$manager = \Drupal::service('plugin.manager.sdc');
foreach (array_keys(\$manager->getDefinitions()) as \$id) {
  echo \$id . PHP_EOL;
}
"

# Vérifier la structure d'un composant (props/slots)
drush php:eval "
\$manager = \Drupal::service('plugin.manager.sdc');
\$component = \$manager->find('mon_theme:card');
print_r(\$component->metadata->schema['properties'] ?? []);
"
```
