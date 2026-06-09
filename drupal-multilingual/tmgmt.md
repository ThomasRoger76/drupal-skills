---
name: drupal-multilingual — TMGMT
description: Translation Management Tool (TMGMT) pour Drupal - workflow de traduction, connecteurs (DeepL, Gengo, local), gestion des jobs de traduction, et automatisation.
---

# TMGMT — Translation Management Tool

## Présentation

TMGMT est le module de gestion des workflows de traduction pour Drupal. Il permet de :
- Envoyer du contenu à des prestataires de traduction
- Recevoir les traductions automatiquement
- Intégrer des services de traduction automatique (DeepL, Google Translate)
- Gérer les workflows de révision et validation

---

## Installation

```bash
# Module TMGMT + connecteurs
composer require drupal/tmgmt
docker compose exec php drush en tmgmt tmgmt_content tmgmt_locale tmgmt_file -y

# Connecteurs de traducteurs (choisir selon le prestataire)
composer require drupal/tmgmt_deepl       # DeepL (MT automatique)
composer require drupal/tmgmt_google      # Google Translate
composer require drupal/tmgmt_gengo       # Gengo (traducteurs humains)
```

---

## Configurer un Traducteur

### Traducteur Local (Tests et Dev)

```
/admin/tmgmt/translators/add

→ Machine translation plugin : Local translator
→ Ce traducteur copie le texte source dans la cible → pour les tests
```

### Traducteur DeepL

```yaml
# config/install/tmgmt.translator.deepl.yml
langcode: fr
status: true
id: deepl
label: DeepL
plugin: tmgmt_deepl
settings:
  deepl_api_key: ''           # Configurer via Key module ou settings.php
  api_version: free           # 'free' ou 'pro'
  max_characters_per_request: 128000
  deepl_tag_handling: html    # html ou xml
```

```php
// settings.php — ne jamais mettre la clé API en YAML
$config['tmgmt.translator.deepl']['settings']['deepl_api_key'] = getenv('DEEPL_API_KEY');
```

---

## Créer un Job de Traduction

### Via l'UI

```
1. Naviguer vers le nœud à traduire
2. Onglet "Translate" (activé par TMGMT)
3. Cocher les langues cibles
4. Cliquer "Request translation"
5. Choisir le traducteur (DeepL, Local...)
6. Soumettre le job
```

### Via PHP

```php
use Drupal\tmgmt\JobInterface;

// Créer un job de traduction pour un nœud
$job = tmgmt_job_create('fr', 'en', 0, [
  'label' => 'Traduction article 42',
]);

// Ajouter le nœud au job
$node = \Drupal\node\Entity\Node::load(42);
$job->addItem('content', 'node', $node->id());

// Configurer le traducteur (champ entité — utiliser set(), pas l'accès direct)
$job->set('translator', 'deepl');
$job->set('settings', []);

// Sauvegarder et soumettre
$job->save();

// Soumettre au traducteur
try {
  $job->requestTranslation();
  \Drupal::messenger()->addStatus(t('Traduction soumise à DeepL.'));
}
catch (\Exception $e) {
  \Drupal::logger('tmgmt')->error('Erreur TMGMT: @error', ['@error' => $e->getMessage()]);
}
```

---

## Workflow Complet

```
1. Éditeur crée un nœud en FR (langue source)
         ↓
2. Éditeur clique "Translate" → sélectionne EN comme cible
         ↓
3. TMGMT crée un "Job" (tmgmt_job)
         ↓
4. Le job est envoyé au traducteur (DeepL, Gengo...)
         ↓
5. La traduction revient (webhook ou polling)
         ↓
6. TMGMT crée automatiquement la traduction EN du nœud
         ↓
7. Réviseur valide et publie la traduction

États d'un job :
  - CREATED : créé, pas encore soumis
  - ACTIVE : soumis au traducteur, en attente
  - CONTINUOUS : soumission automatique
  - ABORTED : annulé
  - FINISHED : traduction reçue et acceptée
```

---

## Suivi des Jobs

```bash
# Lister tous les jobs de traduction
docker compose exec php drush php:eval "
\$jobs = \Drupal::entityTypeManager()->getStorage('tmgmt_job')->loadMultiple();
foreach (\$jobs as \$job) {
  echo \$job->id() . ': ' . \$job->label() . ' [' . \$job->getState() . ']' . PHP_EOL;
  echo '  ' . \$job->getSourceLangcode() . ' → ' . \$job->getTargetLangcode() . PHP_EOL;
}
"

# Jobs en attente (ACTIVE)
docker compose exec php drush php:eval "
\$jobs = \Drupal::entityTypeManager()->getStorage('tmgmt_job')
  ->loadByProperties(['state' => \Drupal\tmgmt\JobInterface::STATE_ACTIVE]);
echo count(\$jobs) . ' job(s) en attente de traduction.' . PHP_EOL;
"

# Vérifier les jobs via l'UI
# /admin/tmgmt/jobs
```

---

## Automatisation — Traduction Continue

```php
// Automatiser la soumission des traductions lors de la publication
function mon_module_node_update(\Drupal\node\NodeInterface $node): void {
  // Si le nœud vient d'être publié → soumettre la traduction automatiquement
  if ($node->isPublished() && !$node->original->isPublished()) {
    $languages_to_translate = ['en', 'de'];  // Langues cibles

    foreach ($languages_to_translate as $target_langcode) {
      if (!$node->hasTranslation($target_langcode)) {
        // Créer un job de traduction automatique
        $job = tmgmt_job_create(
          $node->language()->getId(),
          $target_langcode,
          0  // UID 0 = système
        );
        $job->addItem('content', 'node', $node->id());
        $job->set('translator', 'deepl');
        $job->save();

        try {
          $job->requestTranslation();
        }
        catch (\Exception $e) {
          \Drupal::logger('mon_module')->error(
            'Erreur traduction auto nœud @nid vers @lang: @error',
            ['@nid' => $node->id(), '@lang' => $target_langcode, '@error' => $e->getMessage()]
          );
        }
      }
    }
  }
}
```

---

## Interface de Révision

```
/admin/tmgmt/jobs/{job_id}

Permet de :
  - Voir les segments source et traduit côte à côte
  - Modifier manuellement la traduction avant acceptation
  - Rejeter des segments (renvoyer au traducteur)
  - Accepter et sauvegarder la traduction dans Drupal
```

---

## Commandes Drush TMGMT

```bash
# Lister les traducteurs configurés
docker compose exec php drush php:eval "
\$translators = \Drupal::entityTypeManager()->getStorage('tmgmt_translator')->loadMultiple();
foreach (\$translators as \$t) {
  echo \$t->id() . ': ' . \$t->label() . ' (' . \$t->getPluginId() . ')' . PHP_EOL;
}
"

# Nettoyer les anciens jobs terminés
docker compose exec php drush php:eval "
\$old_jobs = \Drupal::entityTypeManager()->getStorage('tmgmt_job')
  ->loadByProperties(['state' => \Drupal\tmgmt\JobInterface::STATE_FINISHED]);
foreach (\$old_jobs as \$job) {
  if (\$job->getChangedTime() < strtotime('-3 months')) {
    \$job->delete();
  }
}
echo 'Anciens jobs supprimés.';
"
```
