# Leçons — drupal-search

Incidents Search API découverts en projet. Mis à jour après chaque résolution.

---

## 2026-05-16 — Création du skill

### Nouveaux nœuds n'apparaissent pas dans les résultats
- **Symptôme :** Un article publié n'apparaît dans les résultats de recherche qu'après plusieurs heures
- **Cause :** L'indexation automatique via cron n'était pas configurée — `index_directly: false` dans l'index
- **Correct :** Configurer `index_directly: true` dans l'index pour une indexation immédiate après save(), ou configurer le cron correctement
- **Prévention :** Vérifier le statut d'indexation après publication : `drush sapi-s`

### `drush sapi-c && drush sapi-i` non lancé après changement de schéma
- **Symptôme :** Des filtres de facettes ne retournent pas les bonnes valeurs
- **Cause :** Après avoir modifié les champs indexés, l'index n'a pas été vidé et réindexé
- **Correct :** Après tout changement de config de l'index (ajout de champ, processeur...) : `drush sapi-c articles_index && drush sapi-i articles_index`
- **Prévention :** Règle : config index → `drush sapi-c && drush sapi-i` → tester

### Facette affichant des valeurs à 0 résultats
- **Symptôme :** Les éditeurs voient des valeurs de facettes avec "(0)" à côté — lien non fonctionnel
- **Cause :** Le processeur `count_limit` n'était pas configuré avec `minimum_count: 1`
- **Correct :** Ajouter le processeur `count_limit` avec `minimum_count: 1` sur chaque facette
- **Prévention :** Sur toutes les facettes : activer count_limit (min=1) → les valeurs sans résultats disparaissent

### Backend Database en production → recherche ultra-lente
- **Symptôme :** La recherche prend 5-10 secondes sur 50 000 nœuds
- **Cause :** Le backend Database utilise des LIKE SQL — pas de relevance, pas d'optimisation
- **Correct :** Migrer vers Solr : `composer require drupal/search_api_solr` + configurer le serveur Solr + réindexer
- **Prévention :** Backend Database = développement uniquement. Solr obligatoire pour > 5000 entités en production

### Schéma Solr non régénéré après ajout de champ
- **Symptôme :** Nouveau champ indexé dans Search API mais absent des facettes Solr
- **Cause :** Le schéma Solr (schema.xml) n'a pas été régénéré et reployé après l'ajout du champ
- **Correct :** `drush solr-gsc` → copier le schéma → redémarrer le core Solr → `drush sapi-c && drush sapi-i`
- **Prévention :** Checklist ajout de champ Search API + Solr : config → solr-gsc → restart Solr → réindex

### Content Access processor manquant → brouillons dans les résultats
- **Symptôme :** Des articles non publiés (status=0) apparaissent dans les résultats de recherche
- **Cause :** Le processeur `content_access` n'était pas activé dans l'index
- **Correct :** Index → Processors → Activer "Content access" → réindexer
- **Prévention :** Toujours activer `content_access` comme premier processeur sur tout index public — c'est une règle de sécurité

### Views Search sans AJAX → rechargement complet de page à chaque filtre
- **Symptôme :** Chaque modification d'un filtre de recherche recharge toute la page
- **Cause :** `use_ajax: false` dans la configuration de la View Search
- **Correct :** Views → Advanced → Use AJAX → Yes → `drush cr`
- **Prévention :** Toute Vue de recherche doit avoir AJAX activé pour une UX correcte avec les facettes

### Facette liée à la mauvaise source Views
- **Symptôme :** Les facettes n'affectent pas les résultats — filtre semble ignoré
- **Cause :** `facet_source_id` dans la config facette ne correspond pas à l'ID réel de la View
- **Correct :** Format exact : `search_api:views_{display_type}__{view_id}__{display_id}` — vérifier avec `drush config:get facets.facet.ma_facette`
- **Prévention :** Copier l'ID de source depuis l'UI Facets (onglet Facet source dans la config de la Vue)

---

## 2026-06-09 — Audit qualité (v1.1)

### Processor custom converti en attribut PHP → plugin introuvable
- **Symptôme :** Après réécriture d'un processor avec `#[SearchApiProcessor(...)]`, le plugin disparaît de la liste des processors de l'index
- **Cause :** Le module Search API n'a pas migré vers les attributs PHP — la classe `Drupal\search_api\Attribute\SearchApiProcessor` n'existe pas. Seule l'annotation Doctrine `@SearchApiProcessor` est découverte.
- **Correct :** Garder le docblock `/** @SearchApiProcessor(...) */`. Idem pour backends, datasources, trackers custom.
- **Prévention :** Ne pas appliquer mécaniquement « D11 = attributs » aux plugins contrib. Vérifier l'existence de la classe `Attribute` avant de convertir.

### Backend Elasticsearch : `elasticsearch_connector` ET `search_api_opensearch` activés ensemble
- **Symptôme :** Conflit de mapping, serveur Search API qui ne se connecte plus
- **Cause :** Les deux modules fournissent des backends concurrents (`elasticsearch` vs `opensearch`)
- **Correct :** Choisir UN seul module. `elasticsearch_connector` 8.x pour ES/OpenSearch standard, `search_api_opensearch` pour AWS OpenSearch managé.
- **Prévention :** Un seul backend ES/OpenSearch par projet. La connexion d'`elasticsearch_connector` 8.x est portée par le serveur Search API (plus d'entité « cluster » séparée).

### URL Solr non quotée dans un curl shell
- **Symptôme :** `curl ...?action=RELOAD&core=drupal` ne recharge rien, le shell lance un job en arrière-plan sur `core=drupal`
- **Cause :** Le `&` non échappé est interprété par le shell
- **Correct :** Toujours quoter les URLs contenant `&` ou `*` : `curl "http://.../cores?action=RELOAD&core=drupal"`
- **Prévention :** Réflexe : toute URL avec paramètres entre guillemets doubles.
