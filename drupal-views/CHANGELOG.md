# Changelog — drupal-views

---

## v1.1 — 2026-06-09

**Revue D11-currency & conformité standards (lead Drupal)**

### Corrigé
- **Attributs PHP Views (D11) :** tous les exemples de plugins basculés des annotations vers les attributs PHP, conformément au core 11 qui a migré 100 % de ses plugins Views.
  - `custom-handlers.md` — Field/Filter/Sort/Relationship/Area : `@ViewsX` → `#[ViewsX]` + `use Drupal\views\Attribute\...`
  - `views-plugins.md` — Style/Display/Row : annotations + `@Translation` → attributs + `new TranslatableMarkup(...)` (syntaxe core exacte)
  - `views-rest-export.md` — `@RestResource` → `#[RestResource]`
  - `views-paragraphs.md` — `@ViewsField` → `#[ViewsField]`
- **Injection de dépendances :** ajout du pattern `create()` / `ContainerFactoryPluginInterface` pour les handlers et `ResourceBase` ; remplacement des `\Drupal::entityTypeManager()` dans les exemples par des services injectés (règle : pas de `\Drupal::` dans les classes).
- **Display plugin iCal :** suppression du `getRoutedDisplay(): bool` trompeur (la méthode retourne l'ID du display routé) ; route via `uses_route: TRUE`.
- **AJAX Views :** snippet "AJAX custom" réécrit en vanilla JS (l'ancien mélangeait `$(document)` jQuery dans une closure sans `$`, en contradiction avec la règle no-jQuery D10+).
- **Docker natif :** agent `views-generator` — commandes drush préfixées `docker compose exec php` (jamais ddev) ; correction de l'import config erroné (`cim --source=config/install` → flux `cr` + `cex`). Note de convention ajoutée dans `SKILL.md`.

### Ajouté
- `SKILL.md` — note "D11 attributs PHP first" + 2 anti-patterns (DI, annotation legacy) + ligne versioning "plugins core migrés en attributs".
- `views-programmatic.md` — méthode `executeDisplay()` (build + execute + render en un appel) et `preview()`.
- `lessons.md` — 3 leçons (attributs vs annotations, DI dans plugins, `getRoutedDisplay`).

### Vérifié (context7 / source core 11.x)
- Existence des 12 classes `Drupal\views\Attribute\*` + `rest\Attribute\RestResource`.
- Plugins core (`field/Standard`, `style/Grid`) confirmés en attributs avec `TranslatableMarkup`.
- API conservées telles quelles car correctes : `$view->get_total_rows` / `total_rows`, `executeDisplay()`, `getRoutedDisplay()` (signature).

---

## v1.0 — 2026-05-16

**Création initiale**

### Couverture

**`SKILL.md`**
- Quick Decision Table (30+ entrées couvrant UI, hooks, handlers, templates, REST, cache)
- Les règles fondamentales (Views first, code second)
- Anti-patterns critiques (10 entrées avec impact)
- Table versioning D8→D11 (jQuery supprimé D10, PHP attributes D11)

**`hook-views-data.md`**
- `hook_views_data()` complet avec toutes les clés possibles
- Tous les Handler IDs natifs (Fields, Filters, Sorts, Arguments)
- `hook_views_data_alter()` pour modifier des tables existantes
- Tables non-base (join uniquement)
- Organisation du code (mon_module.views.inc)

**`custom-handlers.md`**
- FieldHandler avec `render()`, `preRender()`, `query()`, `defineOptions()`, `buildOptionsForm()`
- FilterHandler avec modification de la WHERE clause
- SortHandler avec expression SQL calculée
- RelationshipHandler
- AreaHandler (header/footer/empty)
- Cache dans les handlers (`getCacheTags()`, `getCacheContexts()`)

**`views-programmatic.md`**
- `Views::getView()` avec pattern de vérification obligatoire
- `hook_views_query_alter()` — modifier la query SQL
- `hook_views_pre_build()` — filtres pré-remplis
- `hook_views_post_execute()` — modifier les résultats
- `hook_views_pre_render()` — cache tags, attachments
- Débogage query SQL (`dpq()`)
- Compter les résultats sans rendu

**`views-templates.md`**
- Règle nommage tirets vs underscores (piège universel)
- Hiérarchie complète des suggestions
- Variables disponibles dans chaque template
- Ajout de suggestions via PHP
- Récapitulatif variables par template

**`views-ajax.md`**
- Activation AJAX sur une View
- Better Exposed Filters (BEF) — widgets disponibles
- Auto-submit sans bouton
- `hook_form_views_exposed_form_alter()` pour personnaliser
- Contextual filters — configuration et PHP programmatique

**`lessons.md`**
- 8 pièges réels documentés avec corrections et prévention

---

## Compatibilité Drupal

| Skill version | Drupal testé | Notes |
|--------------|-------------|-------|
| v1.0 | D9, D10, D11 | jQuery absent D10+, PHP Attributes D11 |
| v1.1 | D10.2+, D11 | Plugins en attributs PHP (standard core), DI via create(), Docker natif |
