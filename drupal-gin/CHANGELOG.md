# Changelog — drupal-gin

---

## v1.2 — 2026-06-08 (audit précision technique D11)

**Corrections factuelles :**
- Levée de la confusion `gin_toolbar` (contrib) vs module **core `navigation`** (D10.3+) vs `gin_login` (login only) — gin-navigation.md, SKILL.md, gin-setup.md
- Valeurs réelles de `classic_toolbar` : `horizontal | vertical | new` (suppression des `sidebar`/`condensed` inventés)
- Clés `gin.settings` corrigées : `enable_darkmode` (0/1/2), `preset_accent_color` + `accent_color`, suppression de `preset`/`content_form_width`/`secondary_toolbar_enabled` douteux
- Suppression de `drush config:set system.theme login gin_login` (gin_login est un module, pas un thème)
- `hook_menu_local_tasks_alter` retiré comme moyen de personnaliser le module Navigation core (faux)
- Signature `hook_form_..._alter` typée (`FormStateInterface`, `string $form_id`) + placement dans `.theme`
- Note cache context `user.permissions` pour le masquage d'items toolbar par rôle
- gin_login.settings.yml : retrait des clés inventées, renvoi vers `drush config:get`
- Migration Adminimal/Seven : ordre correct (activer Gin avant désinstaller), nuance sur la réécriture CSS
- 2 leçons ajoutées (confusion navigation, clés YAML inventées)

---

## v1.1 — 2026-05-16 (audit complet)

**Corrections :**
- See Also mis à jour (drupal-tooling remplacé par drupal-deployment)
- Leçons enrichies (4 leçons au total)
- Fichiers manquants créés (liens QDT résolus)

---

## v1.0 — 2026-05-16

**Création initiale**

- SKILL.md avec Quick Decision Table (2 fichiers de référence)
- lessons.md avec incidents réels
