# Graphify — Knowledge Graph Interactif

## Pourquoi Graphify sur un Vault Drupal

Le Graph View d'Obsidian est interactif mais limité : pas d'export HTML, pas de métriques de centralité, pas de rapport automatisé. Le skill `graphify` transforme le vault Obsidian en un **knowledge graph HTML interactif** avec :
- Communautés détectées automatiquement (clustering)
- Métriques de centralité (quels nœuds sont les plus critiques)
- Export HTML autonome (partageable)
- Rapport d'audit JSON

---

## Pré-requis — Préparer le Vault pour Graphify

### Vérifier que les wikilinks sont cohérents

```bash
# Depuis le dossier du vault Obsidian
# Lister tous les wikilinks — version portable (Linux ET macOS sans GNU grep)
grep -oh '\[\[[^]]*\]\]' $(find . -name "*.md") | sort | uniq -c | sort -rn | head -30

# Version Linux uniquement (avec GNU grep -oP)
# find . -name "*.md" -exec grep -oP '\[\[([^\]]+)\]\]' {} \; | sort | uniq -c | sort -rn | head -30
```

### Filtrer avant de donner à graphify

Graphify sur 200+ notes produit un graphe illisible. Stratégies de filtrage :

```bash
# Option 1 : Graphify uniquement sur les entités
# Copier dans un dossier temporaire
cp -r Entities/ /tmp/drupal-graphify/Entities/

# Option 2 : Graphify sur un sous-ensemble thématique
# Ex: "Article et tout ce qui lui est lié"
cp Entities/ContentTypes/article.md /tmp/drupal-graphify/
cp Fields/field_image.md /tmp/drupal-graphify/
cp Theme/Templates/node--article.html.twig.md /tmp/drupal-graphify/
cp Logic/Hooks/hook_preprocess_node.md /tmp/drupal-graphify/
```

---

## Lancer Graphify

Le skill `graphify` prend n'importe quel input et génère un knowledge graph.

```
/graphify
```

Puis fournir le chemin du vault ou coller le contenu des notes clés.

### Ce que graphify va faire avec le vault Drupal :

1. **Extraire les nœuds** : chaque note `.md` = un nœud du graphe
2. **Extraire les arêtes** : chaque `[[wikilink]]` = une relation entre nœuds
3. **Détecter les communautés** : groupes de nœuds densément liés
4. **Calculer la centralité** : quels nœuds ont le plus de connexions
5. **Générer le HTML** : graphe interactif explorable
6. **Émettre un rapport** : JSON avec métriques + anomalies

---

## Interpréter le Knowledge Graph Drupal

### Communautés attendues dans un projet Drupal

```
Communauté 1 — "Cluster Article"
  ● article.md (hub central)
  ├── field_image, field_tags, field_body
  ├── node--article.html.twig
  ├── node--article--teaser.html.twig
  ├── hook_preprocess_node
  └── mes_articles (View)

Communauté 2 — "Cluster Paragraphs"
  ● hero, card, slider, text_block
  ├── field_image (partagé avec cluster 1 → pont inter-communautés)
  ├── paragraph--hero.html.twig
  └── hook_preprocess_paragraph

Communauté 3 — "Cluster Taxonomies"
  ● tags, categories
  ├── field_tags (pont → Article)
  └── field_categorie (pont → Projet)
```

**Les nœuds entre communautés (ponts) = points critiques** — si `field_image` est dans 2 communautés, toute modification impact les deux.

### Métriques à surveiller dans le rapport graphify

```json
{
  "nodes_count": 87,
  "edges_count": 234,
  "communities": [
    {
      "id": 0,
      "size": 15,
      "label": "Cluster Article",
      "hub_node": "article"
    }
  ],
  "high_centrality_nodes": [
    {"node": "field_image", "degree": 12, "betweenness": 0.85},
    {"node": "hook_preprocess_node", "degree": 8, "betweenness": 0.72}
  ],
  "isolated_nodes": ["display.node.page.teaser", "field_obsolete"],
  "anomalies": [
    "hero paragraph has no Twig template note",
    "field_couleur has 0 backlinks (orphan field?)"
  ]
}
```

**high_centrality_nodes** → Champs et hooks critiques à documenter en priorité  
**isolated_nodes** → Notes orphelines à vérifier (entités non documentées ?)  
**anomalies** → Lacunes d'architecture à corriger

---

## Canvas Obsidian — Flux de Données Config → Twig

En complément du Knowledge Graph, le Canvas Obsidian permet de dessiner le **flux de données**.

### Template Canvas : Flux Article complet

```json
{
  "nodes": [
    {
      "id": "db", "type": "text",
      "text": "## Base de Données\n`node_field_data`\n`node__field_image`",
      "x": -600, "y": 0, "width": 200, "height": 120,
      "color": "4"
    },
    {
      "id": "config", "type": "file",
      "file": "Entities/ContentTypes/article.md",
      "x": -300, "y": 0, "width": 250, "height": 120
    },
    {
      "id": "fields", "type": "text",
      "text": "## Champs\n[[field_image]]\n[[field_tags]]\n[[field_body]]",
      "x": 0, "y": 0, "width": 200, "height": 120,
      "color": "2"
    },
    {
      "id": "display", "type": "file",
      "file": "display.node.article.default.md",
      "x": 300, "y": 0, "width": 250, "height": 120
    },
    {
      "id": "preprocess", "type": "file",
      "file": "Logic/Hooks/hook_preprocess_node.md",
      "x": 300, "y": -200, "width": 250, "height": 100,
      "color": "5"
    },
    {
      "id": "twig", "type": "file",
      "file": "Theme/Templates/node--article.html.twig.md",
      "x": 600, "y": 0, "width": 250, "height": 120,
      "color": "3"
    },
    {
      "id": "html", "type": "text",
      "text": "## HTML Final\n`<article class=\"node--article\">`",
      "x": 900, "y": 0, "width": 200, "height": 80,
      "color": "1"
    }
  ],
  "edges": [
    {"id": "e1", "fromNode": "db", "toNode": "config", "label": "Entity API"},
    {"id": "e2", "fromNode": "config", "toNode": "fields", "label": "field definitions"},
    {"id": "e3", "fromNode": "fields", "toNode": "display", "label": "formatters"},
    {"id": "e4", "fromNode": "display", "toNode": "twig", "label": "render array"},
    {"id": "e5", "fromNode": "preprocess", "toNode": "twig", "label": "$variables"},
    {"id": "e6", "fromNode": "twig", "toNode": "html", "label": "rendu"}
  ]
}
```

Sauvegarder ce fichier en `.canvas` dans `Dashboard/Architecture Overview.canvas`.

---

## Workflow Complet — Drupal → Graphify → Insights

```bash
# 1. Exporter la config Drupal
docker compose exec php drush cex -y

# 2. Générer le vault Obsidian
python3 scripts/drupal_to_obsidian.py \
  --config config/sync \
  --vault ~/Obsidian/MonProjet

# 3. Ouvrir dans Obsidian → vérifier les backlinks + Graph View

# 4. Lancer graphify sur les dossiers clés
# → /graphify sur les notes Entities/ + Fields/ + Theme/

# 5. Analyser le rapport JSON de graphify
# → Identifier high_centrality_nodes et anomalies

# 6. Mettre à jour la documentation selon les findings
# → Ajouter les templates manquants
# → Documenter les champs orphelins

# 7. Commit du vault dans le dépôt du projet (optionnel)
git add docs/obsidian/
git commit -m "docs: mise à jour du vault architecture"
```

---

## Utiliser graphify sur des Sous-Ensembles Thématiques

### "Impact d'une migration de champ"

Si `field_image` doit changer de type :
1. Extraire toutes les notes liées : `find . -name "*.md" -exec grep -l "field_image" {} \;`
2. Copier ces notes dans un dossier temporaire
3. Lancer graphify → voir le périmètre d'impact visuel

### "Cartographie d'un Paragraph complexe"

Pour documenter le Paragraph `hero` avant refactoring :
1. `hero.md` + tous ses champs + son template + ses preprocess
2. Graphify → rapport des connexions = périmètre de régression
