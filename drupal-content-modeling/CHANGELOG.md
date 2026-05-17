# Changelog — drupal-content-modeling

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
| v1.0 | D8, D9, D10, D11 | Paragraphs contrib toutes versions, Layout Builder stable D8.7+, Media Library core D9+ |
