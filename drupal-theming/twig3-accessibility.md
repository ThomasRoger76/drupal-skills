# Twig 3 & Accessibilité — Drupal 11

## Section A — Twig 3 (standard D11)

### 1. Arrow functions dans les filtres

```twig
{# |map avec arrow function #}
{% set noms = utilisateurs|map(u => u.name) %}
{% set urls = nodes|map(n => path('entity.node.canonical', {node: n.id})) %}

{# |filter avec condition #}
{% set publies = articles|filter(a => a.isPublished) %}
{% set recents = items|filter(i => i.created > (now - 86400)) %}

{# |reduce pour aggregation #}
{% set total = prices|reduce((carry, v) => carry + v, 0) %}

{# Chaîner les opérations #}
{% set titres_actifs = nodes|filter(n => n.isPublished)|map(n => n.label) %}
```

> Syntaxe Twig 3 : `u => u.name` (sans parenthèses pour un seul argument).
> Twig 2 nécessitait une closure PHP complète — la syntaxe arrow est native en Twig 3.

### 2. apply block — remplace les filtres sur blocs

```twig
{# Transformer un bloc de contenu #}
{% apply upper %}
  {{ 'titre'|t }}
{% endapply %}

{% apply trim|lower %}
  {{ content.field_description }}
{% endapply %}
```

> `{% apply %}` est l'équivalent Twig 3 de `{% filter %}` (deprecated). Chaîner avec `|`.

### 3. has_value check (Drupal + Twig 3)

```twig
{# Vérifier si un champ Drupal a une valeur #}
{% if content.field_image|render is not empty %}
  {{ content.field_image }}
{% endif %}

{# Vérifier une valeur de champ brute #}
{% if node.field_subtitle.value %}
  <p class="subtitle">{{ node.field_subtitle.value }}</p>
{% endif %}
```

---

## Section B — Accessibilité WCAG 2.1 AA dans les templates Twig

### 1. Navigation avec ARIA

```twig
<nav aria-label="{{ 'Navigation principale'|t }}" role="navigation">
  <ul>
    {% for item in items %}
      <li{{ item.attributes.addClass(item.is_expanded ? 'menu__item--expanded' : '') }}>
        <a href="{{ item.url }}"
           {# in_active_trail = page courante ou ancêtre dans le menu (variable réelle du menu Drupal) #}
           {% if item.in_active_trail %}aria-current="page"{% endif %}
           {% if item.below %}
             aria-expanded="{{ item.in_active_trail ? 'true' : 'false' }}"
             aria-haspopup="true"
           {% endif %}>
          {{ item.title }}
        </a>
        {% if item.below %}
          <ul aria-label="{{ item.title }}">
            {% for child in item.below %}
              <li><a href="{{ child.url }}">{{ child.title }}</a></li>
            {% endfor %}
          </ul>
        {% endif %}
      </li>
    {% endfor %}
  </ul>
</nav>
```

**Points clés :**
- `aria-current="page"` sur l'item actif (pas `aria-selected`)
- `aria-expanded` sur les items avec sous-menu — valeur dynamique selon `in_active_trail`
- `aria-haspopup="true"` indique un sous-menu
- `aria-label` sur les `<ul>` imbriqués pour distinguer les niveaux

### 2. Images — alt obligatoire et contextuel

```twig
{# Image décorative — alt vide + role presentation #}
<img src="{{ decorative_image }}" alt="" role="presentation">

{# Image informative — utiliser le champ Drupal (alt géré dans le media/image) #}
{{ content.field_image }}

{# Image avec alt forcé depuis preprocess PHP #}
{# Dans mon_theme.theme : #}
{# $variables['content']['field_image'][0]['#item']->set('alt', $custom_alt); #}

{# Image de lien — décrire la destination, pas l'image #}
<a href="{{ url }}">
  <img src="{{ icon }}" alt="{{ 'Voir le profil de @name'|t({'@name': user.name}) }}">
</a>
```

**Règle WCAG 1.1.1 :** toute image `<img>` doit avoir un attribut `alt`. Vide (`alt=""`) pour les décorations, descriptif pour les images porteuses de sens.

### 3. Formulaires accessibles

```twig
{# Forms API Drupal génère automatiquement les associations label/input #}
{# Pour les templates custom ou les widgets complexes : #}
<div class="form-item">
  <label for="field-{{ element['#id'] }}">
    {{ element['#title'] }}
    {% if element['#required'] %}
      <span class="form-required" aria-hidden="true">*</span>
    {% endif %}
  </label>
  <input
    id="field-{{ element['#id'] }}"
    type="{{ element['#type'] }}"
    name="{{ element['#name'] }}"
    aria-required="{{ element['#required'] ? 'true' : 'false' }}"
    {% if element['#description'] %}
      aria-describedby="desc-{{ element['#id'] }}"
    {% endif %}
  >
  {% if element['#description'] %}
    <div id="desc-{{ element['#id'] }}" class="form-item__description">
      {{ element['#description'] }}
    </div>
  {% endif %}
  {% if element['#errors'] %}
    <div class="form-item__error-message" role="alert">
      {{ element['#errors'] }}
    </div>
  {% endif %}
</div>
```

### 4. Skip links (accessibilité navigation clavier)

```twig
{# En PREMIER dans html.html.twig, avant tout autre contenu #}
<a href="#main-content" class="visually-hidden focusable skip-link">
  {{ 'Aller au contenu principal'|t }}
</a>
<a href="#navigation" class="visually-hidden focusable skip-link">
  {{ 'Aller à la navigation'|t }}
</a>

{# CSS requis dans le thème : #}
{# .visually-hidden { position: absolute; width: 1px; height: 1px; overflow: hidden; clip: rect(0,0,0,0); } #}
{# .visually-hidden.focusable:focus { position: static; width: auto; height: auto; clip: auto; } #}

{# La cible dans page.html.twig : #}
<main id="main-content" tabindex="-1">
  {{ page.content }}
</main>
```

> Drupal core (`stable9`, `olivero`) fournit `visually-hidden` et `focusable` dans ses CSS. Utiliser ces classes directement.

### 5. Messages d'erreur accessibles

```twig
{# Messages flash (statut, avertissement, erreur) #}
{% for type, messages_list in messages %}
  <div
    role="{{ type == 'error' ? 'alert' : 'status' }}"
    aria-live="{{ type == 'error' ? 'assertive' : 'polite' }}"
    class="messages messages--{{ type }}"
  >
    <ul>
      {% for message in messages_list %}
        <li>{{ message }}</li>
      {% endfor %}
    </ul>
  </div>
{% endfor %}
```

**Règles `aria-live` :**
- `assertive` = interrompt le lecteur d'écran immédiatement (erreurs critiques)
- `polite` = attend la fin de la lecture en cours (confirmations, infos)

### 6. Tableaux accessibles

```twig
<table aria-label="{{ 'Liste des articles'|t }}">
  <caption>{{ 'Articles publiés'|t }}</caption>
  <thead>
    <tr>
      <th scope="col" id="col-titre">{{ 'Titre'|t }}</th>
      <th scope="col" id="col-date">{{ 'Date'|t }}</th>
      <th scope="col" id="col-auteur">{{ 'Auteur'|t }}</th>
    </tr>
  </thead>
  <tbody>
    {% for row in rows %}
      <tr>
        <td headers="col-titre">{{ row.titre }}</td>
        <td headers="col-date">
          <time datetime="{{ row.date|date('Y-m-d') }}">{{ row.date|date('d/m/Y') }}</time>
        </td>
        <td headers="col-auteur">{{ row.auteur }}</td>
      </tr>
    {% endfor %}
  </tbody>
</table>
```

**Points clés :**
- `<caption>` décrit le tableau (équivaut à un titre pour les lecteurs d'écran)
- `scope="col"` ou `scope="row"` sur les `<th>`
- `<time datetime="...">` pour les dates machine-readable
- `aria-label` complète `<caption>` quand le contexte n'est pas évident

### 7. Anti-patterns accessibilité

| ❌ À ne jamais faire | ✅ Bonne pratique | Critère WCAG |
|---------------------|------------------|--------------|
| `<div onclick="...">` pour un bouton | `<button type="button">` | 4.1.2 Nom, rôle, valeur |
| `<img>` sans attribut `alt` | `alt=""` (décoratif) ou alt descriptif | 1.1.1 Contenu non textuel |
| `<a href="#">Cliquer ici</a>` | `<a href="...">Lire l'article sur [sujet]</a>` | 2.4.6 En-têtes et étiquettes |
| `<input>` sans `<label>` associé | `<label for="id">` ou `aria-label` | 1.3.1 Info et relations |
| Couleur seule pour véhiculer une info | Couleur + icône + texte | 1.4.1 Utilisation de la couleur |
| Contraste texte < 4.5:1 (normal) ou < 3:1 (large) | Vérifier avec axe DevTools | 1.4.3 Contraste |
| `tabindex="2"` ou valeurs positives | `tabindex="0"` ou `tabindex="-1"` uniquement | 2.4.3 Ordre de focus |
| Ouvrir lien dans nouvel onglet sans avertir | Ajouter `({{ 'nouvel onglet'|t }})` ou icône + `aria-label` | 3.2.2 À la saisie |
| `<table>` pour la mise en page | CSS Grid/Flexbox | 1.3.1 Info et relations |
| Focus invisible (outline: none) | Conserver ou améliorer l'outline navigateur | 2.4.7 Focus visible |
