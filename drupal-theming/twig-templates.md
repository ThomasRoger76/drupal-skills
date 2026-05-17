# Twig — Templates et Filtres Drupal

## Syntaxe de Base

```twig
{# Afficher une variable (auto-escape XSS) #}
{{ variable }}

{# Commentaire (non rendu en HTML) #}
{# Ceci est ignoré à l'exécution #}

{# Structures de contrôle #}
{% if node.bundle == 'article' %}
  <div class="article">{{ content }}</div>
{% elseif node.bundle == 'page' %}
  <div class="page">{{ content }}</div>
{% else %}
  {{ content }}
{% endif %}

{% for item in items %}
  <li>{{ item.content }}</li>
{% else %}
  <li>Aucun item.</li>
{% endfor %}

{# Assigner une variable locale #}
{% set ma_classe = node.isPromoted() ? 'promoted' : '' %}

{# Affichage brut — DANGEREUX si source inconnue #}
{{ variable|raw }}
```

---

## Filtres Drupal Indispensables

```twig
{# TRADUCTION — toujours utiliser |t pour les strings UI #}
{{ 'Bonjour'|t }}
{{ 'Bonjour @name'|t({'@name': user.name}) }}

{# Placeholders dans |t :
   @var → htmlencoded (valeurs utilisateur, texte brut sécurisé)
   %var → htmlencoded ET encadré en <em>...</em>
   :var → transformé en <a href="url">url</a> cliquable (génère du HTML, pas juste une URL)
#}
{{ 'Soumis par @author le @date'|t({'@author': author_name, '@date': date}) }}

{# RENDER — forcer le rendu d'un render array (ex: pour un test avec |render) #}
{{ content.field_image|render }}
{% if content.field_image|render %}
  <div class="has-image">{{ content.field_image }}</div>
{% endif %}

{# WITHOUT — afficher content en excluant certains champs #}
{{ content|without('field_image', 'field_tags') }}
{# Très courant pour afficher le reste du contenu après avoir placé certains champs manuellement #}

{# CLEAN_CLASS — nettoyer une string pour usage en attribut class #}
{{ node.bundle|clean_class }}               {# 'my bundle' → 'my-bundle' #}
{{ view_mode|clean_class }}                 {# 'Full Content' → 'full-content' #}

{# CLEAN_ID — nettoyer pour usage en attribut id #}
{{ 'Mon Titre'|clean_id }}                  {# 'mon-titre' #}

{# SAFE_JOIN — joindre des éléments de manière sûre #}
{{ classes|safe_join(' ') }}

{# DATE — formater un timestamp Unix #}
{{ node.created.value|date('d/m/Y') }}
{{ node.created.value|date('F j, Y', 'Europe/Paris') }}

{# CHECK_MARKUP — appliquer un format de texte (filter format) #}
{{ body_value|check_markup(body_format) }}

{# STRIPTAGS — retirer le HTML #}
{{ node.body.value|striptags }}

{# TRIM — retirer espaces début/fin #}
{{ variable|trim }}

{# FORMAT_SIZE — formater une taille de fichier #}
{{ file.filesize|format_size }}   {# '2.4 MB' #}
```

---

## Fonctions Drupal dans Twig

```twig
{# Charger une librairie CSS/JS #}
{{ attach_library('mon_theme/modal') }}

{# Générer une URL absolue vers une route #}
<a href="{{ url('entity.node.canonical', {'node': node.id}) }}">Lire plus</a>

{# Générer un chemin relatif #}
<a href="{{ path('mon_module.liste') }}">Liste</a>

{# URL d'un fichier géré (managed file) #}
<img src="{{ file_url(node.field_image.entity.uri.value) }}" alt="">

{# Créer un objet d'attributs de toutes pièces #}
{% set mon_attr = create_attribute() %}
{{ mon_attr.addClass('ma-classe').setAttribute('data-id', node.id) }}

{# Tester si on est sur la page d'accueil (variable injectée par template_preprocess_page) #}
{% if is_front %}...{% endif %}

{# Debug — afficher le contenu complet d'une variable (Twig debug actif requis) #}
{{ dump() }}           {# Dump TOUT le contexte Twig courant (toutes les variables) #}
{{ dump(content) }}    {# Dump une variable spécifique #}
{{ dump(attributes) }}
```

---

## L'Objet `attributes` — Critique

**Règle absolue : toujours utiliser les méthodes, jamais de string directe.**

```twig
{# ❌ FAUX — écrase les classes Drupal système (block--myid, etc.) #}
<div class="{{ attributes.class }} ma-classe">

{# ✅ CORRECT — ajoute sans écraser #}
<div{{ attributes.addClass('ma-classe') }}>

{# Méthodes disponibles #}
{{ attributes.addClass('classe-a', 'classe-b') }}
{{ attributes.removeClass('classe-a') }}
{{ attributes.setAttribute('data-id', node.id) }}
{{ attributes.removeAttribute('data-id') }}
{{ attributes.hasClass('node--promoted') }}    {# retourne true/false #}
{{ attributes }}                                 {# affiche tous les attributs #}

{# Ajouter une classe conditionnellement #}
<article{{ attributes.addClass(
  'node',
  'node--type-' ~ node.bundle|clean_class,
  node.isPromoted() ? 'node--promoted',
  node.isSticky() ? 'node--sticky',
  not node.isPublished() ? 'node--unpublished',
  view_mode ? 'node--view-mode-' ~ view_mode|clean_class
) }}>

{# title_attributes et content_attributes existent aussi #}
<h2{{ title_attributes.addClass('node__title') }}>{{ label }}</h2>
<div{{ content_attributes.addClass('node__content') }}>{{ content }}</div>
```

---

## Accéder aux Données Drupal

```twig
{# content = render array du contenu (applique les formatters) #}
{{ content.field_image }}           {# Affiche le champ avec son formatter #}
{{ content.field_body }}
{{ content }}                        {# Affiche tous les champs #}
{{ content|without('field_tags') }}  {# Tous sauf field_tags #}

{# node = entité brute (accès aux valeurs directes) #}
{{ node.label }}                     {# Titre du nœud #}
{{ node.bundle }}                    {# Type de contenu #}
{{ node.id }}                        {# NID #}
{{ node.created.value }}             {# Timestamp de création #}
{{ node.uid.entity.name.value }}     {# Nom de l'auteur #}
{{ node.field_image.entity.uri.value }}  {# URI du fichier image #}
{{ node.field_image.alt }}           {# Alt text de l'image #}

{# Champs multi-valeurs — itérer #}
{% for item in node.field_tags %}
  <span>{{ item.entity.label }}</span>
{% endfor %}

{# Champs référence — accéder à l'entité référencée #}
{{ node.field_categorie.entity.label }}
{{ node.field_auteur.entity.field_bio.value }}

{# Variables de page standard disponibles dans page.html.twig #}
{{ page.header }}         {# Région header #}
{{ page.content }}        {# Région contenu #}
{{ page.sidebar_first }}  {# Région sidebar — conditionner avant affichage #}
{% if page.sidebar_first %}
  <aside class="sidebar">{{ page.sidebar_first }}</aside>
{% endif %}
```

---

## Templates Principaux — Structures de Base

### `page.html.twig`

```twig
<div class="layout-container">
  <header role="banner">
    {{ page.header }}
    {% if page.primary_menu %}
      <nav role="navigation" aria-label="{{ 'Site navigation'|t }}">
        {{ page.primary_menu }}
      </nav>
    {% endif %}
  </header>

  {% if page.breadcrumb %}{{ page.breadcrumb }}{% endif %}
  {% if page.highlighted %}
    <div class="highlighted">{{ page.highlighted }}</div>
  {% endif %}

  <main role="main" id="main-content" tabindex="-1">
    <div class="layout-content">
      {{ page.content }}
    </div>
    {% if page.sidebar_first %}
      <aside class="sidebar sidebar--first" role="complementary">
        {{ page.sidebar_first }}
      </aside>
    {% endif %}
  </main>

  {% if page.footer %}
    <footer role="contentinfo">{{ page.footer }}</footer>
  {% endif %}
</div>
```

### `node.html.twig`

```twig
<article{{ attributes.addClass('node', 'node--type-' ~ node.bundle|clean_class,
  node.isPromoted() ? 'node--promoted',
  not node.isPublished() ? 'node--unpublished',
  view_mode ? 'node--view-mode-' ~ view_mode|clean_class)
}}>

  {{ title_prefix }}
  {% if not page %}
    <h2{{ title_attributes.addClass('node__title') }}>
      <a href="{{ url }}" rel="bookmark">{{ label }}</a>
    </h2>
  {% endif %}
  {{ title_suffix }}

  {% if display_submitted %}
    <footer class="node__meta">
      {{ author_picture }}
      <span{{ author_attributes }}>
        {% trans %}By {{ author_name }} on {{ date }}{% endtrans %}
      </span>
    </footer>
  {% endif %}

  <div{{ content_attributes.addClass('node__content') }}>
    {{ content|without('comment') }}  {# Afficher tout sauf les commentaires #}
  </div>

</article>
```

### `block.html.twig`

```twig
<div{{ attributes.addClass('block', 'block-' ~ configuration.provider|clean_class, 'block-' ~ plugin_id|clean_class) }}>
  {{ title_prefix }}
  {% if label %}
    <h2{{ title_attributes }}>{{ label }}</h2>
  {% endif %}
  {{ title_suffix }}
  {% block content %}
    {{ content }}
  {% endblock %}
</div>
```

### `field.html.twig`

```twig
{# field.html.twig — template de base pour tous les champs #}
<div{{ attributes.addClass(
  'field',
  'field--name-' ~ field_name|clean_class,
  'field--type-' ~ field_type|clean_class,
  'field--label-' ~ label_display
) }}>

  {# Label du champ #}
  {% if label_display != 'hidden' and label_display != 'visually_hidden' %}
    <div{{ title_attributes.addClass('field__label') }}>{{ label }}</div>
  {% elseif label_display == 'visually_hidden' %}
    <div{{ title_attributes.addClass('field__label', 'visually-hidden') }}>{{ label }}</div>
  {% endif %}

  {# Items du champ #}
  {% if multiple %}
    <div class="field__items">
      {% for item in items %}
        <div{{ item.attributes.addClass('field__item') }}>{{ item.content }}</div>
      {% endfor %}
    </div>
  {% else %}
    {% for item in items %}
      <div{{ item.attributes.addClass('field__item') }}>{{ item.content }}</div>
    {% endfor %}
  {% endif %}

</div>
```

---

## Héritage de Templates

```twig
{# templates/content/node--article--full.html.twig #}
{# Étendre le template parent (cherche dans la chaîne de base themes) #}
{% extends "node.html.twig" %}

{% block content %}
  {# Modifier uniquement la section content #}
  <div class="article-layout">
    <div class="article-main">
      {{ content|without('field_sidebar') }}
    </div>
    {% if content.field_sidebar|render %}
      <aside class="article-sidebar">
        {{ content.field_sidebar }}
      </aside>
    {% endif %}
  </div>
{% endblock %}
```

---

## Fonctions Utiles — Patterns Courants

```twig
{# Intégrer une View directement dans un template #}
{{ drupal_view('nom_de_la_view', 'identifiant_display') }}
{{ drupal_view('articles_recents', 'block_1') }}

{# Intégrer un bloc par plugin ID #}
{{ drupal_block('system_branding_block') }}

{# Afficher les liens du nœud (commentaires, partage, flags…) #}
{{ content.links }}

{# Afficher tout le contenu SAUF les liens et les tags #}
{{ content|without('links', 'field_tags') }}

{# Générer un lien vers une entité #}
<a href="{{ node.toUrl().toString() }}">{{ node.label }}</a>

{# Vérifier qu'un champ a du contenu avant affichage #}
{% if content.field_image|render %}
  <div class="featured-image">{{ content.field_image }}</div>
{% endif %}
```

---

**`{% trans %}` pour les translations multi-lignes :**

```twig
{% trans %}
  Soumis par {{ author_name }} le {{ date }}.
{% endtrans %}

{# Avec pluriel #}
{% set count = items|length %}
{% trans %}
  {{ count }} article
{% plural count %}
  {{ count }} articles
{% endtrans %}
```
