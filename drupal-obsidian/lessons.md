# Leçons — drupal-obsidian

Problèmes rencontrés lors du mapping Drupal → Obsidian. Mis à jour après chaque incident.

---

## 2026-05-14 — Création du skill

### Frontmatter YAML incompatible avec Dataview — Requêtes vides
- **Symptôme :** Les requêtes Dataview retournent 0 résultats malgré des notes présentes
- **Cause :** Le frontmatter généré par le script Python contient des guillemets ou des caractères spéciaux qui cassent le parsing YAML d'Obsidian (ex: `label: "L'article"` → apostrophe non échappée)
- **Correct :** Toujours utiliser des guillemets doubles et échapper les caractères spéciaux. Dans le script Python : `label = label.replace("'", "\\'")` OU utiliser le style bloc YAML
- **Prévention :** Tester les requêtes Dataview sur 5-10 notes avant de faire l'extraction complète

### Wikilinks vers notes inexistantes — Faux orphelins dans Graph View
- **Symptôme :** De nombreux nœuds gris (notes non créées) dans le Graph View, alourdissant la visualisation
- **Cause :** Les templates de notes créent des liens vers des notes qui n'ont pas encore été créées (ex: `[[display.node.article.default]]` si la note de mode d'affichage n'a pas été extraite)
- **Correct :** Dans Graph View → Filtres → Désactiver "Afficher les notes orphelines" TANT QUE toutes les notes ne sont pas créées. Ou créer des notes vides placeholder
- **Prévention :** Créer d'abord les notes "cibles" (champs, modes d'affichage) avant les entités qui les lient

### Graphify sur tout le vault — Graphe illisible (200+ nœuds)
- **Symptôme :** Le graphe graphify est une boule de nœuds impossible à lire
- **Cause :** Trop de notes incluant les dashboards, les requêtes Dataview, etc.
- **Correct :** Toujours pré-filtrer : copier uniquement `Entities/ + Fields/` dans un dossier temporaire avant de lancer graphify
- **Prévention :** Ne jamais lancer graphify sur l'ensemble du vault — utiliser des sous-ensembles thématiques (≤ 50 notes)

### Script Python : encodage YAML avec caractères spéciaux
- **Symptôme :** `yaml.safe_dump` génère des `\n` littéraux dans les notes au lieu de sauts de ligne
- **Cause :** `yaml.dump` par défaut utilise `\n` comme séquence d'échappement
- **Correct :** Utiliser `yaml.dump(data, default_flow_style=False, allow_unicode=True)` + `encoding='utf-8'` sur le `write_text`
- **Prévention :** Tester le script sur 3-4 fichiers YAML avant de lancer l'extraction complète

### Dataview : `file.outlinks` vs `file.inlinks` confondus
- **Symptôme :** La requête "Paragraphs sans template Twig" retourne tous les Paragraphs
- **Cause :** Confusion entre `file.outlinks` (liens sortants depuis la note) et `file.inlinks` (backlinks entrants)
- **Correct :** 
  - `file.outlinks` = ce que CETTE note référence (`[[autre-note]]`)  
  - `file.inlinks` = qui référence CETTE note
  - Pour "bundles utilisant ce champ" → `file.inlinks` sur la note du champ
- **Prévention :** Tester chaque requête Dataview avec `TABLE file.outlinks, file.inlinks FROM "Entities"` pour valider

### Vault non synchronisé avec la config Drupal
- **Symptôme :** Des champs ou entités dans le vault n'existent plus en config Drupal réelle
- **Cause :** Régénération du vault après `drush cex` sans effacer l'ancien vault → notes obsolètes persistent
- **Correct :** Avant une ré-extraction, archiver l'ancien vault : `mv ~/Obsidian/MonProjet ~/Obsidian/MonProjet.old` puis extraire proprement
- **Prévention :** Intégrer la régénération dans le workflow de déploiement : `drush cex && python3 drupal_to_obsidian.py`

### Notes de template Twig — Convention unifiée `tpl-` (DÉCISION FINALE)
- **Symptôme :** Obsidian peut mal gérer les doubles extensions (`.html.twig.md`) selon l'OS ou la version
- **Convention adoptée :** `tpl-{template-name}.md` — préfixe `tpl-`, tirets, SANS `.html.twig`
  - `node--article.html.twig` → note `tpl-node--article.md` → wikilink `[[tpl-node--article]]`
  - `paragraph--hero.html.twig` → note `tpl-paragraph--hero.md` → wikilink `[[tpl-paragraph--hero]]`
- **Le script d'extraction génère automatiquement** les bons wikilinks avec `tpl_link()` et `drupal_id_to_twig_filename()`
- **Prévention :** Ne jamais inclure `.html.twig` dans le nom de fichier Obsidian

### `_build_fields_table` retourne `?` pour tous les types de champs
- **Symptôme :** La table des champs affiche `?` dans la colonne Type pour tous les champs
- **Cause :** `field.field.*` (instances) ne contient PAS `field_type` — le type est dans `field.storage.*`
- **Correct :** Le script doit croiser les instances avec les storages. La fonction `_build_fields_table(fields, all_storages)` accepte maintenant `all_storages` comme second argument
- **Prévention :** Toujours passer le dict `storages` retourné par `extract_field_storages()` aux fonctions d'extraction des entités

### `grep -oP` incompatible macOS — erreur silencieuse
- **Symptôme :** La commande grep retourne 0 résultats ou une erreur sur macOS
- **Cause :** `-P` (Perl regex) n'est pas supporté par le BSD grep de macOS
- **Correct :** Utiliser `grep -oh '\[\[[^]]*\]\]'` (POSIX, portable Linux+macOS)
- **Prévention :** Toujours tester les commandes grep avec les deux variantes ou utiliser `ripgrep` (`rg`)
