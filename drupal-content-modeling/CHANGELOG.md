# Changelog — drupal-content-modeling

---

## v1.1 — 2026-06-09

**Revue qualité — corrections factuelles et conformité standards**

### Corrigé
- **`paragraphs.md`** — Erreur factuelle : le module Paragraphs n'a PAS d'attribut PHP `#[ParagraphsBehavior]` (y compris D11). Suppression de la mention trompeuse « D11 : utiliser #[ParagraphsBehavior] » et de l'import inutile `use ...Annotation\ParagraphsBehavior`. Note explicite : conserver l'annotation `@ParagraphsBehavior`.
- **`paragraphs.md`** — Bug Twig : ternaire sans `else` (`isPublished() ? ...`) injectait `false` dans le tableau de classes. Remplacé par une classe inconditionnelle + gestion `unpublished` + `|filter(c => c is not empty)`.
- **`custom-entities.md`** — Setting invalide `text_processing` sur un champ `string` → remplacé par `->setSetting('max_length', 64)`.
- **`custom-entities.md`** — Bloc D11/D8-D10 clarifié : l'attribut `#[ContentEntityType]` est désigné comme voie par défaut en D11 (avec `use` requis et rappel qu'un attribut ne vit pas dans un docblock), annotation `@ContentEntityType` présentée comme legacy. Avertissement « choisir UNE seule déclaration ».
- **`SKILL.md`** — Table décisionnelle cassée : 11 lignes « modules contrib » à 3 colonnes mélangées dans une table à 4 colonnes (rendu Markdown désaligné). Extraites dans une nouvelle section « Modules Contrib & Approches Utiles » (table 3 colonnes homogène).
- **`SKILL.md`** — `dxpr_builder` (propriétaire) remplacé par `layout_paragraphs` (contrib libre) comme première option drag-and-drop — conforme au principe « contrib d'abord ».
- **`nodes-content-types.md`, `layout-builder.md`** — Toutes les commandes drush préfixées `docker compose exec php drush ...` (Docker natif, jamais ddev).
- **`lessons.md`** — Leçon `#[ParagraphsBehavior]` corrigée + 3 nouvelles leçons (ternaire Twig, `text_processing`, contrib libre vs propriétaire).

---

## v1.0 — 2026-05-16

**Création initiale**

### Couverture

**`SKILL.md`**
- Quick Decision Table (25+ entrées) — Node, Paragraphs, Layout Builder, Custom Entity, Taxonomy, Menu, Block
- Tableau décisionnel Paragraphs vs Layout Builder (8 critères)
- Anti-patterns critiques (10 entrées)
- Table versioning D8→D11 (Layout Builder stable D8.7, Media Library core D9+)

**`paragraphs.md`**
- Architecture Paragraphs (schéma Node → Paragraph → sous-Paragraph)
- Configuration d'un champ Paragraphs via YAML
- Paragraphs Behaviors — `ParagraphsBehaviorBase` complet (buildBehaviorForm, preprocess, settingsSummary, isApplicable)
- Accès programmatique (problème N+1 et solution batch loading)
- Paragraphes imbriqués — pattern correct avec batch loading multi-niveau
- Templates Twig (hiérarchie, variables disponibles, behavior_settings)
- Création programmatique (Paragraph::create + attach to node)
- Performance Varnish & cache avec Paragraphs

**`field-types.md`**
- Tableau de sélection complet (20+ types)
- Texte — String vs text_long vs text_with_summary (arbre décisionnel)
- Date — datetime (date seule) vs datetime (avec heure) vs daterange
- Image vs File vs Media (tableau comparatif)
- Entity Reference — configuration YAML par cible
- List — valeurs fixes (string, integer)
- Cardinality — champs multi-valeurs (appendItem, count, iterate)
- Accès programmatique aux valeurs de chaque type

**`lessons.md`**
- 8 mauvaises décisions architecturales avec conséquences réelles

---

## Compatibilité Drupal

| Skill version | Drupal | Notes |
|--------------|--------|-------|
| v1.1 | D8, D9, D10, D11 | Idem v1.0 + précision : Paragraphs Behaviors restent en annotation `@ParagraphsBehavior` même en D11 ; Custom Entity core utilise l'attribut `#[ContentEntityType]` en D11 |
| v1.0 | D8, D9, D10, D11 | Paragraphs contrib toutes versions, Layout Builder stable D8.7+, Media Library core D9+ |
