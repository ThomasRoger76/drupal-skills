# Changelog — drupal-token

---

## v1.2 — 2026-06-08 (mise à niveau Drupal 11)

**Corrections majeures :**
- `hook_tokens()` : ajout du 5e paramètre obligatoire `BubbleableMetadata` (D8.6+) avec signature complète et `addCacheableDependency()` dans tous les blocs d'exemple — corrige l'omission la plus grave (cache non invalidé).
- Anti-pattern SKILL.md corrigé : `cache.default` → `BubbleableMetadata` (l'ancienne formulation était fausse). Ajout des anti-patterns « oublier BubbleableMetadata » et « HTML brut/XSS ».
- Nouvelle section D11 : hooks OOP par attribut `#[Hook('token_info')]` / `#[Hook('tokens')]` dans `src/Hook/`.
- `module_load_include()` (déprécié D10.3+) remplacé par `.tokens.inc` auto-chargé + `\Drupal::moduleHandler()->loadInclude()`.
- token-basics.md : `validate()` (inexistant) → `getInvalidTokensByContext()` (méthode réelle du service).
- tokens-in-modules.md : template Twig `displayname` corrigé en `commande.uid.entity.displayName`.
- Tableau Évolution par Version enrichi (BubbleableMetadata, `#[Hook]`, `module_load_include`).
- 2 nouvelles leçons (BubbleableMetadata, module_load_include).

---

## v1.1 — 2026-05-16 (audit complet)

**Corrections :**
- See Also mis à jour (drupal-tooling remplacé par drupal-deployment)
- Leçons enrichies (7 leçons au total)
- Fichiers manquants créés (liens QDT résolus)

---

## v1.0 — 2026-05-16

**Création initiale**

- SKILL.md avec Quick Decision Table (3 fichiers de référence)
- lessons.md avec incidents réels
