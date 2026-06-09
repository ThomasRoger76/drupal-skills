# Anatomie d'un Thème Drupal

## Arborescence Complète

```
mon_theme/
├── mon_theme.info.yml           # Métadonnées (obligatoire)
├── mon_theme.libraries.yml      # Bibliothèques CSS/JS
├── mon_theme.theme              # Preprocess + hooks PHP
├── mon_theme.breakpoints.yml    # Points de rupture responsives
├── logo.svg                     # Logo (référencé dans .info.yml)
├── screenshot.png               # Capture admin (/admin/appearance)
├── favicon.ico
├── config/
│   ├── install/                 # Config importée à l'activation du thème
│   └── schema/                  # Schéma de validation config
├── css/
│   ├── base/                    # Reset, variables CSS, typographie de base
│   ├── layout/                  # Grille, régions, structure de page
│   ├── components/              # Boutons, cartes, navigation, formulaires
│   └── theme/                   # Couleurs, polices — surcharges finales
├── js/
│   └── mon-comportement.js
├── images/
├── templates/
│   ├── layout/
│   │   └── page.html.twig
│   ├── content/
│   │   ├── node.html.twig
│   │   └── node--article.html.twig
│   ├── block/
│   ├── field/
│   ├── navigation/
│   ├── views/
│   └── misc/
└── src/                         # PHP optionnel (plugins, EventSubscribers)
```

---

## `.info.yml` — Anatomie Complète

```yaml
# mon_theme.info.yml
name: Mon Thème
description: 'Thème custom pour le projet X.'
type: theme
core_version_requirement: ^10 || ^11
package: Custom

# Héritage — LIRE "Choix du Base Theme" ci-dessous
base theme: stable9

# Régions — déclaration obligatoire si tu veux les utiliser
regions:
  page_top: 'Page top'            # Réservé aux messages système
  header: 'En-tête'
  primary_menu: 'Menu principal'
  secondary_menu: 'Menu secondaire'
  breadcrumb: 'Fil d''Ariane'
  highlighted: 'Mis en avant'
  help: 'Aide'
  content: 'Contenu'              # OBLIGATOIRE — région principale
  sidebar_first: 'Barre latérale gauche'
  sidebar_second: 'Barre latérale droite'
  footer_first: 'Pied de page 1'
  footer_second: 'Pied de page 2'
  page_bottom: 'Page bottom'      # Réservé aux scripts footer

# Librairies chargées GLOBALEMENT sur toutes les pages
libraries:
  - mon_theme/global-styling
  - mon_theme/global-scripts

# Remplacer une librairie core ou contrib par la nôtre
libraries-override:
  core/normalize:
    css:
      base:
        assets/vendor/normalize-css/normalize.css: false   # Désactiver un fichier
  system/base:                    # Désactiver entièrement une librairie
    false

# ÉTENDRE une librairie existante sans la remplacer
libraries-extend:
  core/drupal.dialog:
    - mon_theme/dialog-extend     # Ajouter nos styles au dialog Drupal

# Supprimer des CSS core/contrib individuellement
stylesheets-remove:
  - core/assets/vendor/normalize-css/normalize.css
  - '@mon_module/css/module-styles.css'            # @MODULE_MACHINE_NAME pour cibler un module contrib

# Stylesheet pour l'éditeur CKEditor (édition WYSIWYG)
ckeditor_stylesheets:
  - css/theme/ckeditor.css

# Fichiers logo et favicon (chemins relatifs au thème)
logo: logo.svg
favicon: favicon.ico

# PHP minimum du thème (string exacte, pas de ^)
php: '8.1'

# Masquer ce thème de l'UI admin (utile pour les base themes)
hidden: false
```

---

## Choix du Base Theme

| Base Theme | Quand | Ce qu'il fournit |
|-----------|-------|-----------------|
| `stable9` | Thème custom de zéro sur D9-D11 | Templates minimal stables — peu d'opinion CSS |
| `olivero` | Étendre le thème front D10+ | Design moderne, responsive natif |
| `claro` | Thème d'admin custom | UI admin accessible, système de design cohérent |
| `false` | Contrôle total | Aucun template hérité — tu fournis tout |
| `bootstrap5` | Projets Bootstrap 5 (contrib) | Grille BS5, classes utilitaires — nécessite `drupal/bootstrap5` |
| `gin` | Thème d'admin custom moderne (contrib) | UX admin avancée, sidebar navigation |
| ~~`classy`~~ | ❌ Supprimé D10 | Remplacer par `stable9` ou `false` |
| ~~`bartik`~~ | ❌ Supprimé D10 | Était le thème front D8/D9 |

> **Note Bootstrap5 :** si `base theme: bootstrap5`, la dépendance Composer est obligatoire :
> `composer require drupal/bootstrap5`
> et le thème contrib active jQuery par défaut — `core/jquery` est alors acceptable dans les librairies.

**Héritage :** un thème hérite des templates de son base theme. Si `node.html.twig` n'existe pas dans ton thème, Drupal remonte la chaîne vers le base theme.

---

## Workflow de Développement

### Activer le thème et configurer le debug

```bash
# Activer le thème
docker compose exec php drush config:set system.theme default mon_theme -y
docker compose exec php drush cr

# Activer Twig Debug (jamais en production)
# Copier le fichier services si pas encore fait
cp web/sites/default/default.services.yml web/sites/default/services.yml
```

Éditer `web/sites/default/services.yml` :
```yaml
parameters:
  twig.config:
    debug: true            # Affiche les commentaires de suggestion de template
    auto_reload: true      # Recharge les templates sans drush cr
    cache: false           # Désactive le cache Twig
```

```bash
docker compose exec php drush cr   # OBLIGATOIRE après modification de services.yml
```

**Désactiver l'agrégation CSS/JS** (dev uniquement) :
`/admin/config/development/performance` → décocher les deux cases d'agrégation

### Cycle de développement standard

```bash
# 1. Modifier templates/CSS/JS
# 2. Pour CSS/JS : rafraîchir le navigateur (si agrégation désactivée)
# 3. Pour templates Twig : drush cr ou auto_reload si activé
# 4. Pour preprocess/.theme : TOUJOURS drush cr
docker compose exec php drush cr

# Débugger les templates actifs (cr = alias cache:rebuild, un seul suffit)
docker compose exec php drush cr

# Voir les erreurs PHP du thème
docker compose exec php drush watchdog:tail --type=php
```

---

## Troubleshooting

| Problème | Cause probable | Solution |
|----------|---------------|---------|
| Template non pris en compte | Mauvais nom de fichier ou chemin | Activer Twig debug → copier le nom exact du commentaire HTML |
| CSS/JS absent | Librairie non attachée ou chemin incorrect | Vérifier `.libraries.yml`, désactiver l'agrégation |
| Thème n'apparaît pas dans l'UI | YAML invalide dans `.info.yml` | `drush cr`, puis valider la syntaxe YAML |
| Variables absentes dans Twig | Cache des templates pas vidé après preprocess | `drush cr` — les fonctions preprocess sont mises en cache |
| `base theme: classy` erreur D10 | Module supprimé | Remplacer par `stable9` ou `false` |
| `{{ dump(variable) }}` blanc | Twig debug non activé | Vérifier `services.yml` et `drush cr` |
| Région vide malgré bloc assigné | Région non déclarée dans `.info.yml` | Ajouter la région au YAML |

---

## Single Directory Components (SDC) — D10.3+ / D11 ★ STANDARD D11

Les SDC sont **le système de composants officiel de Drupal 11** — ils remplacent le pattern preprocess+twig fragmenté par un composant auto-contenu (template + CSS + JS + schéma).

### Structure complète d'un SDC

```
mon_theme/components/
  card/
    card.component.yml   # Schéma des props (OBLIGATOIRE)
    card.twig            # Template du composant — DOIT être {id}.twig, PAS .html.twig
    card.css             # CSS scopé au composant
    card.js              # JS comportement Drupal
  hero/
    hero.component.yml
    hero.twig
    hero.css
  button/
    button.component.yml
    button.twig
    button.css
```

> **⚠️ Nommage du template SDC :** le fichier Twig doit s'appeler exactement comme l'ID du
> composant suivi de `.twig` (ex. `card.twig`) — **jamais** `card.html.twig`. Avec l'extension
> `.html.twig`, Drupal ne découvre pas le composant et le rendu `{% component %}` échoue.

### `card.component.yml` — Schéma complet des props

```yaml
$schema: 'https://git.drupalcode.org/project/drupal/-/raw/HEAD/core/assets/schemas/v1/metadata.schema.json'
name: Card
description: Carte article avec image, titre et description.
props:
  type: object
  required:
    - title
  properties:
    title:
      type: string
      title: Titre
    body:
      type: string
      title: Corps de texte
      nullable: true
    image_url:
      type: string
      title: URL de l'image
      nullable: true
    url:
      type: string
      title: Lien de la carte
      nullable: true
    variant:
      type: string
      title: Variante visuelle
      enum: [default, featured, compact]
      default: default
slots:
  footer:
    title: Contenu du pied de carte (optionnel)
```

### `card.twig` — Template du composant

```twig
{#
  @prop title: string
  @prop body: string|null
  @prop image_url: string|null
  @prop url: string|null
  @prop variant: string
  @slot footer
#}
<article class="card card--{{ variant }}">
  {% if image_url %}
    <img src="{{ image_url }}" alt="{{ title }}" loading="lazy">
  {% endif %}

  <div class="card__content">
    {% if url %}
      <h2 class="card__title"><a href="{{ url }}">{{ title }}</a></h2>
    {% else %}
      <h2 class="card__title">{{ title }}</h2>
    {% endif %}

    {% if body %}
      <div class="card__body">{{ body }}</div>
    {% endif %}
  </div>

  {% if footer is defined %}
    <footer class="card__footer">{{ footer }}</footer>
  {% endif %}
</article>
```

### Utiliser un SDC dans un template parent

```twig
{# Dans node--article--teaser.html.twig #}
{% component 'mon_theme:card' with {
  title: node.label,
  body: content.body|render|striptags|trim|slice(0, 150),
  image_url: content.field_image|render,
  url: url,
  variant: 'default'
} %}

{# Avec un slot nommé #}
{% component 'mon_theme:card' with { title: node.label } %}
  {% fill footer %}
    <time>{{ node.created.value|date('d/m/Y') }}</time>
  {% endfill %}
{% endcomponent %}

{# Composer des SDC imbriqués #}
{% component 'mon_theme:hero' with { title: page_title } %}
  {% fill content %}
    {% component 'mon_theme:button' with { label: 'Voir plus', url: url } %}{% endcomponent %}
  {% endfill %}
{% endcomponent %}
```

### Utiliser un SDC depuis un preprocess PHP

```php
// Dans mon_theme.theme — attacher un SDC via render array
function mon_theme_preprocess_node(array &$variables): void {
  $node = $variables['node'];
  // Construire un render array avec le composant SDC
  $variables['card_component'] = [
    '#type' => 'component',
    '#component' => 'mon_theme:card',
    '#props' => [
      'title' => $node->label(),
      'url' => $node->toUrl()->toString(),
      'variant' => 'featured',
    ],
  ];
}
```

### Activer les SDC

Depuis Drupal **10.3**, les SDC sont **stables et intégrés au core** : aucun module à activer
(`drush en sdc` échoue — le composant n'est pas un module installable séparément). Les composants
sont détectés automatiquement dès qu'un dossier `components/` valide existe dans le thème.

```bash
# D10.3+ / D11 : il suffit de vider le cache pour que Drupal découvre les composants
docker compose exec php drush cr
```

> En D10.1 / D10.2, les SDC étaient un module core **expérimental** activable via `drush en sdc -y`.
> Cette étape n'est plus nécessaire (ni possible) à partir de D10.3.

### Debugging SDC

```bash
# Afficher les erreurs de validation des props
docker compose exec php drush cr && docker compose exec php drush watchdog:show --type=sdc

# Lister les composants disponibles
docker compose exec php drush php:eval "print_r(\Drupal::service('plugin.manager.sdc')->getDefinitions());"
```

### Anti-patterns SDC

| ❌ | ✅ | Raison |
|----|----|----|
| Logique PHP complexe dans le `.twig` du SDC | Preprocess + `#props` depuis PHP | SDC = présentation pure |
| Props non typées dans `.component.yml` | Toujours déclarer `type:` + `required:` | Validation au rendu |
| SDC sans slot pour le contenu variable | Utiliser `slots:` pour le contenu dynamique | Composabilité |
| Un seul gros composant `page` | Décomposer : `header`, `card`, `button` | Réutilisabilité |
