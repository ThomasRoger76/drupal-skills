# Changelog — drupal-sdc

---

## v1.2 — 2026-06-08 (audit qualité — correctness API)

**Corrections de fond (API réelle vérifiée via Context7 / core Drupal) :**
- Service SDC corrigé partout : `sdc.component_registry` (inexistant) → `plugin.manager.sdc`
  (`ComponentPluginManager`). 6 commandes `drush php:eval` de debug réparées dans SKILL.md,
  sdc-setup.md, sdc-integration.md, sdc-layout-builder.md.
- Paramètre de validation corrigé : `sdc.debug` (inexistant) → `sdc.enforce_schemas` (core D10.3+),
  dans SKILL.md (4 lignes), sdc-setup.md et lessons.md (2 leçons).
- Leçon « slots » corrigée : `{{ slots.badge }}` était FAUX — slots et props sont des variables
  Twig de premier niveau (`{{ badge }}`), conforme au template `card.twig` et au core
  (`mergeAdditionalRenderContext`).
- Block/Layout plugins migrés vers les attributs PHP D11 (`#[Block]`, `#[Layout]`) au lieu
  des annotations `@Block` / `@Layout` dépréciées.

**Ajouts (complétude patterns natifs) :**
- Section « Surcharger un composant » : `replaceComponent` (info.yml) + `hook_component_info_alter`.
- `libraryOverrides` dans `card.component.yml` (attacher une librairie partagée).
- 2 nouvelles leçons (service, slots) ; table d'évolution par version affinée (D10.1-10.2 / D10.3 / D11).

---

## v1.1 — 2026-05-16 (audit complet)

**Corrections :**
- See Also mis à jour (drupal-tooling remplacé par drupal-deployment)
- Leçons enrichies (4 leçons au total)
- Fichiers manquants créés (liens QDT résolus)

---

## v1.0 — 2026-05-16

**Création initiale**

- SKILL.md avec Quick Decision Table (3 fichiers de référence)
- lessons.md avec incidents réels
