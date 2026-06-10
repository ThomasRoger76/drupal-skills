---
name: drupal-ai
description: Use when integrating AI/LLM capabilities into Drupal with the drupal/ai module ecosystem - installing the AI core framework and provider modules (ai_provider_anthropic, ai_provider_openai, ai_provider_ollama, ai_provider_mistral), storing API keys securely with the key module (never in config), calling chat/embeddings/image operations programmatically via AiProviderPluginManager, enabling the CKEditor 5 AI assistant (ai_ckeditor) for editors, generating alt text or summaries automatically with ai_automators field chains, building RAG search with ai_search + search_api + a vector database backend (pgvector, Milvus, Pinecone), adding a site chatbot (deepchat), wiring AI into ECA workflows, building AI agents that manipulate Drupal config, choosing models per task (cost vs quality), handling GDPR/data-privacy concerns when content leaves the site, or debugging provider errors and token costs in Drupal 10.3-11+
---

# Drupal AI — intégrer les LLM avec l'écosystème drupal/ai

> **Convention d'exécution.** Toutes les commandes s'exécutent en **Docker natif** :
> `docker compose exec php drush …` / `docker compose exec php composer …`. Jamais DDEV.

## Architecture de l'écosystème (module AI 1.x)

Le module **`drupal/ai`** est le framework unifié (successeur des modules `openai_*` éparpillés) :
une API d'abstraction + des **providers** interchangeables + des sous-modules fonctionnels.

| Couche | Modules | Rôle |
|--------|---------|------|
| Cœur | `ai` | Abstraction : operation types (chat, embeddings, moderation, text_to_image, speech_to_text…) |
| Providers | `ai_provider_anthropic`, `ai_provider_openai`, `ai_provider_ollama`, `ai_provider_mistral`, `ai_provider_litellm` | Connecteurs LLM — interchangeables sans toucher au code appelant |
| Clés | `key` | Stockage sécurisé des API keys (env var / fichier) — **jamais dans la config exportée** |
| Éditorial | `ai_ckeditor`, `ai_content_suggestions` | Assistant CKEditor 5, suggestions (résumé, ton, traduction) |
| Automation | `ai_automators` | Chaînes de génération sur champs (alt text, SEO meta, taxonomie auto) |
| Recherche | `ai_search` + `search_api` + VDB (`ai_vdb_provider_postgres` pgvector, Milvus, Pinecone) | Embeddings + RAG |
| Conversationnel | `deepchat` (chatbot), `ai_assistant_api` | Chatbot front + assistants configurables |
| Orchestration | `ai_agents`, ECA + `ai_eca` | Agents qui agissent sur le site, workflows événementiels |

## Installation de base (Anthropic)

```bash
docker compose exec php composer require drupal/ai drupal/ai_provider_anthropic drupal/key
docker compose exec php drush en ai ai_provider_anthropic key -y
```

**Clé API — protocole obligatoire :**
1. Clé dans une **variable d'environnement** (`ANTHROPIC_API_KEY` via `.env` docker, jamais commitée).
2. Créer une Key (`/admin/config/system/keys/add`) de type *Environment variable*.
3. Configurer le provider (`/admin/config/ai/providers`) en pointant cette Key.
4. `drush cex` : la config exportée ne contient que le **nom** de la key — vérifier qu'aucun
   secret n'apparaît dans `config/sync` avant commit.

Définir ensuite les **defaults par operation type** (`/admin/config/ai/settings`) : quel provider
+ modèle pour chat, embeddings, etc. Le code appelant reste agnostique.

## Appel programmatique (chat)

```php
// Service à injecter : ai.provider (AiProviderPluginManager). Version indicative API ai 1.x.
use Drupal\ai\OperationType\Chat\ChatInput;
use Drupal\ai\OperationType\Chat\ChatMessage;

$manager = \Drupal::service('ai.provider'); // injecter via le container en vrai code
$defaults = $manager->getDefaultProviderForOperationType('chat');
$provider = $manager->createInstance($defaults['provider_id']);

$input = new ChatInput([
  new ChatMessage('user', 'Résume ce texte en 2 phrases : ' . $texte),
]);
$response = $provider->chat($input, $defaults['model_id'], ['mymodule']);
$resume = $response->getNormalized()->getText();
```

Règles :
- **Toujours passer par l'abstraction** `ai.provider` — jamais le SDK Anthropic/OpenAI en direct
  dans un module métier (sinon le provider n'est plus interchangeable ni configurable).
- Tags (3ᵉ argument) = traçabilité des appels dans les logs AI.
- Opérations longues → **QueueWorker** (`drupal-cron-queue`), jamais dans le thread de requête.

## Choix de modèle par tâche (coût vs qualité)

| Tâche | Modèle recommandé | Pourquoi |
|-------|-------------------|----------|
| Alt text, tags, classification | Haiku / petit modèle | Volume élevé, faible complexité |
| Résumés, suggestions éditoriales | Sonnet / modèle moyen | Bon ratio qualité/prix |
| Agents, génération structurée complexe | Opus / grand modèle | Fiabilité du raisonnement |
| Embeddings RAG | Modèle d'embeddings dédié du provider | Jamais un modèle de chat |

## RAG / recherche sémantique (ai_search)

1. `composer require drupal/ai_search` + un VDB provider (ex. `ai_vdb_provider_postgres` → pgvector).
2. Créer un **serveur Search API** de type AI Search (choix VDB + modèle d'embeddings).
3. Index : champs à vectoriser (title + body rendus) + **chunking** (taille/overlap dans l'index).
4. Vue ou assistant branché sur l'index → réponses avec contexte (RAG).

Pièges : ré-indexer après changement de modèle d'embeddings (vecteurs incompatibles) ;
`drush sapi-i` en file pour les gros volumes ; pgvector nécessite l'extension PostgreSQL.

## Automation éditoriale (ai_automators)

Cas type — alt text automatique : champ image → onglet *Automator* → chaîne
« image → description » sur le champ alt, déclenchée à la sauvegarde si vide.
Autres patterns : meta description SEO, extraction d'entités vers taxonomie, traduction de brouillon.
Toujours laisser l'éditeur **réviser** (workflow brouillon), jamais publier de l'IA brute.

## Sécurité & RGPD (non négociable en projet client)

- **Données sortantes** : tout appel envoie du contenu au provider — documenter dans le registre
  de traitement ; pour les données sensibles, provider on-premise (`ai_provider_ollama`).
- **Clés** : module `key` + env vars. Une clé dans `config/sync` ou en DB claire = incident.
- **Logs** : activer le logging AI en dev, le restreindre en prod (les prompts peuvent contenir
  des données personnelles).
- **Coûts** : poser des quotas/alertes côté provider ; un automator mal configuré sur un
  `hook_entity_presave` peut générer des milliers d'appels.
- **Injection de prompt** : ne jamais concaténer du contenu utilisateur anonyme dans un prompt
  d'agent ayant des capacités d'action (création de config, etc.).

## Debugging

```bash
docker compose exec php drush watchdog:show --filter=ai --count=20
```
- Erreur 401/403 → key mal résolue (env var absente du container — vérifier `docker compose exec php printenv`).
- Réponses vides/tronquées → `max_tokens` du modèle dans la config provider.
- Latence → streaming (si le contexte le permet) ou passage en queue.

## See Also

- `drupal-cron-queue` — QueueWorker pour les appels AI asynchrones
- `drupal-search` — Search API, la base sur laquelle ai_search s'appuie
- `drupal-security` — clés, données sensibles, permissions
- `claude-api` (skill global) — référence API Anthropic si provider custom nécessaire
