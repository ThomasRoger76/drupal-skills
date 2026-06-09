# Leçons — drupal-content-modeling

Mauvaises décisions architecturales et leurs conséquences. Mis à jour après chaque projet.

---

## 2026-05-16 — Création du skill

### Paragraphs imbriqués trop profonds — formulaire inutilisable
- **Symptôme :** Les éditeurs se plaignent que le formulaire d'édition est trop complexe et lent
- **Cause :** 4 niveaux d'imbrication (page → section → colonne → bloc → sous-bloc) — le formulaire inline Paragraphs charge tout en mémoire
- **Correct :** Aplatir à 2-3 niveaux maximum. Utiliser le mode "modal" pour les paragraphes profonds
- **Prévention :** Règle : max 3 niveaux de nesting. Si besoin de plus → reconsidérer l'architecture

### N+1 queries Paragraphs en preprocess — page ultra-lente
- **Symptôme :** Page avec 10 paragraphes = 11+ requêtes DB — temps de génération > 2 secondes
- **Cause :** `$item->entity` dans foreach sans batch loading — une requête par paragraphe
- **Correct :** `loadMultiple()` avec tous les IDs collectés avant d'itérer
- **Prévention :** Toujours utiliser `loadMultiple()` sur les champs Paragraphs — jamais `->entity` en boucle

### Migrer de Paragraphs vers Layout Builder — irréversible et coûteux
- **Symptôme :** Après 2 ans avec Paragraphs, l'équipe veut Layout Builder — migration de 500 nœuds nécessaire
- **Cause :** Décision initiale non documentée — Paragraphs choisi par habitude sans évaluer Layout Builder
- **Correct :** Aucun fix facile — migration de contenu complexe avec perte potentielle de structure
- **Prévention :** Prendre la décision Paragraphs vs Layout Builder EN DÉBUT DE PROJET — c'est presque irréversible

### Taxonomy utilisée comme système de navigation
- **Symptôme :** La navigation du site est lente et difficile à gérer — les URLs de taxonomy ne correspondent pas aux UX attendues
- **Cause :** Le développeur a utilisé un vocabulaire Taxonomy comme système de navigation ("Accueil > Produits > Smartphones")
- **Correct :** Utiliser les Menu links Drupal pour la navigation — Taxonomy pour la classification
- **Prévention :** Taxonomy = classification de contenu. Navigation = Menu links. Les deux ont des APIs distinctes.

### Champ Body (WYSIWYG) pour du texte sans HTML
- **Symptôme :** Surcharge inutile de format texte — possibilité d'injection HTML non intentionnelle
- **Cause :** `field_description` créé en type `text_with_summary` alors que c'est juste une description courte
- **Correct :** `string_long` pour du texte sans HTML — `text_long` avec Basic HTML si du balisage minime est nécessaire
- **Prévention :** Question à se poser : "L'éditeur a-t-il vraiment besoin de <strong>, <ul>, <a> ici ?" → Non → String

### Entity Reference sans index → View ultra-lente
- **Symptôme :** Views listant des nœuds filtrés par auteur : 15 secondes pour s'afficher
- **Cause :** `node_field_data.uid` sans index, JOIN sur `users_field_data` non indexé
- **Correct :** Vérifier les indexes sur les colonnes de jointure : `SHOW INDEX FROM node_field_data`
- **Prévention :** Toute colonne utilisée en JOIN ou WHERE fréquemment doit avoir un index

### Custom entity utilisée quand un Node suffit
- **Symptôme :** Développement d'un système complexe de custom entity pour des "articles de blog"
- **Cause :** Le développeur voulait "quelque chose de propre" sans surcharge Drupal
- **Correct :** Nodes sont parfaits pour le contenu SEO avec URL, révisions, traduction, workflow
- **Prévention :** Custom entity = données métier sans nécessité de frontend/SEO. Contenu = Node.

### Paragraphs Behaviors — `#[ParagraphsBehavior]` non découvert (y compris D11)
- **Symptôme :** Le Behavior plugin n'apparaît pas dans l'interface Paragraphs
- **Cause :** Conversion de l'annotation `@ParagraphsBehavior` en attribut PHP `#[ParagraphsBehavior]`. Contrairement au core Drupal, le module Paragraphs n'a PAS migré ses plugins Behavior vers les attributs — même en D11 le manager attend l'annotation.
- **Correct :** Conserver l'annotation `@ParagraphsBehavior(...)` dans le docblock, toutes versions D8→D11.
- **Prévention :** Ne pas généraliser « D11 = attributs PHP ». Le core supporte `#[...]` ; chaque module contrib migre à son rythme. Vérifier le PluginManager du module avant de convertir.

---

## 2026-06-09 — Revue qualité

### Ternaire Twig sans `else` dans un tableau de classes → classe `false`
- **Symptôme :** Une classe parasite (ou un crash `clean_class` sur un booléen) apparaît dans le `class` du paragraphe non publié.
- **Cause :** `paragraph.isPublished() ? 'paragraph--view-mode--' ~ view_mode` injecte le booléen `false` dans le tableau quand la condition est fausse — Twig n'a pas de branche `else`.
- **Correct :** Sortir la classe inconditionnelle, gérer l'état non publié à part, et filtrer : `[...]|filter(c => c is not empty)`.
- **Prévention :** Dans un tableau de classes Twig, soit fournir les deux branches du ternaire, soit filtrer les valeurs vides en sortie.

### Setting `text_processing` sur un champ `string` (BaseFieldDefinition)
- **Symptôme :** Setting ignoré / confusion sur la nature du champ.
- **Cause :** `text_processing` ne s'applique qu'aux champs texte formatés (`text`, `text_long`, `text_with_summary`), pas à `string`/`string_long`.
- **Correct :** Pour un `string`, seul `max_length` est pertinent : `->setSetting('max_length', 64)`.
- **Prévention :** Ne pas copier les settings d'un type de champ vers un autre. `string` = brut sans format texte.

### « Contrib d'abord » : ne pas recommander un module propriétaire en première option
- **Symptôme :** Tableau de décision suggérait `dxpr_builder` (propriétaire/payant) pour le drag-and-drop éditeur.
- **Correct :** Recommander `layout_paragraphs` (contrib libre et mature) en première intention ; mentionner les solutions propriétaires seulement si un besoin spécifique le justifie.
- **Prévention :** Première option = core ou contrib libre. Propriétaire = exception documentée, jamais le défaut.
