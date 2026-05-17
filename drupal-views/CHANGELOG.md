# Changelog — drupal-views

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
