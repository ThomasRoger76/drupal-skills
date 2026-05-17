---
name: drupal-token — basics
description: Tokens natifs Drupal - catalogue complet, syntaxe, chaining, utilisation dans les configurations, et remplacement programmatique.
---

# Token Basics — Catalogue et Utilisation

## Installer le Module Token

```bash
composer require drupal/token
drush en token -y

# L'interface Token Tree est disponible sur :
# /admin/help/token
# (et dans tous les champs qui supportent les tokens, via "Browse available tokens")
```

---

## Tokens Natifs — Référence

### Tokens Node (`[node:*]`)

```
[node:title]                          → Titre du nœud
[node:url]                            → URL absolue
[node:url:relative]                   → URL relative
[node:body]                           → Corps complet (HTML)
[node:body:summary]                   → Résumé automatique (300 chars)
[node:summary]                        → Champ résumé si disponible
[node:created]                        → Date de création (format par défaut)
[node:created:long]                   → Date format "long" Drupal
[node:created:html_datetime]          → ISO 8601 (pour Schema.org)
[node:changed]                        → Date de modification
[node:author:name]                    → Nom de l'auteur
[node:author:mail]                    → Email de l'auteur
[node:type]                           → Machine name du type (article, page)
[node:type:name]                      → Label du type (Article, Page)
[node:nid]                            → ID du nœud
[node:uuid]                           → UUID du nœud
[node:langcode]                       → Code langue (fr, en)

# Champs custom
[node:field_mon_champ]                → Valeur du champ
[node:field_tags:0:name]              → Premier terme du champ tags
[node:field_tags:join]                → Tous les termes séparés par virgule
[node:field_image:entity:file:url]    → URL du fichier image
[node:field_ref:entity:title]         → Titre de l'entité référencée
```

### Tokens Site (`[site:*]`)

```
[site:name]                           → Nom du site
[site:slogan]                         → Slogan
[site:url]                            → URL racine
[site:mail]                           → Email admin
[site:login-url]                      → URL de connexion
```

### Tokens User (`[user:*]`, `[current-user:*]`)

```
[user:uid]                            → ID utilisateur
[user:name]                           → Nom d'utilisateur
[user:mail]                           → Email
[user:url]                            → URL du profil
[current-user:name]                   → Utilisateur courant
[current-user:mail]
[current-user:uid]
[current-user:field_prenom]           → Champ profil custom
```

### Tokens Date (`[date:*]`)

```
[date:short]                          → Date courte (aujourd'hui)
[date:medium]
[date:long]
[date:custom:Y-m-d]                  → Format custom PHP
[date:html_datetime]                  → ISO 8601
```

### Tokens Page Courante

```
[current-page:title]                  → Titre de la page courante
[current-page:url]                    → URL de la page courante
[current-page:url:relative]
```

---

## Token Chaining — Valeurs par Défaut

```
# Syntaxe : [token1|token2|valeur_fixe]
# Si token1 est vide → essayer token2 → sinon utiliser valeur_fixe

[node:summary|node:body:summary|Aucun résumé disponible]

# Exemples Metatag :
[node:title|current-page:title]      → Titre du nœud OU titre de la page
[node:field_og_image:entity:file:url|site:logo:url]  → Image custom OU logo du site
```

---

## Remplacement Programmatique

```php
// Remplacer les tokens dans une string
$token_service = \Drupal::service('token');

// Avec contexte d'une entité
$node = \Drupal\node\Entity\Node::load(42);
$text = 'Bonjour depuis [node:title] sur [site:name]';

$replaced = $token_service->replace($text, ['node' => $node]);
// → 'Bonjour depuis Mon Article sur Mon Site'

// Sans contexte (tokens site/date disponibles)
$replaced = $token_service->replace('[site:name] — [date:custom:Y]');

// Nettoyer les tokens non résolus (éviter [node:field_optionnel] dans le texte)
$replaced = $token_service->replace($text, ['node' => $node], ['clear' => TRUE]);

// Valider les tokens dans un texte (vérifier qu'ils sont syntaxiquement corrects)
$errors = $token_service->validate('[node:CHAMP_INEXISTANT]', ['node']);
// → ['Invalid token: [node:CHAMP_INEXISTANT]']

// Obtenir la liste des tokens disponibles pour un type
$info = $token_service->getInfo();
$node_tokens = array_keys($info['tokens']['node'] ?? []);
```

---

## Token Browser dans les Formulaires PHP (Forms API)

```php
// Ajouter le Token Browser à un champ textfield custom
// (le bouton "Browse available tokens" apparaît sous le champ)
$form['mon_texte_avec_tokens'] = [
  '#type' => 'textarea',
  '#title' => $this->t('Message avec tokens'),
  '#default_value' => $config['message'] ?? '',
  '#description' => $this->t('Utiliser des tokens comme [node:title] ou [user:name].'),
  // Clé pour activer le Token Tree browser
  '#token_types' => ['node', 'user', 'site'],
];

// Attacher le module token pour le browser UI
if (\Drupal::moduleHandler()->moduleExists('token')) {
  $form['token_tree'] = [
    '#theme' => 'token_tree_link',      // Lien compact "Browse tokens"
    '#token_types' => ['node', 'user'],
    '#show_restricted' => TRUE,         // Inclure les tokens restreints
    '#weight' => 90,
  ];
}

// OU afficher l'arbre complet inline
$form['token_tree_full'] = [
  '#theme' => 'token_tree_table',      // Tableau complet des tokens
  '#token_types' => ['node'],
  '#show_nested' => FALSE,
  '#weight' => 91,
];
```

**Template `token_tree_link` :** affiche un lien cliquable "Browse available tokens" qui ouvre une modale. Nécessite le module `token` activé.

---

## Tokens dans les Configurations UI

```
Modules qui utilisent les tokens :
  ├── Metatag    → [node:title], [node:body:summary], [node:field_image:entity:file:url]
  ├── Pathauto   → [node:title], [node:type], [node:field_categorie:entity:name]
  ├── Webform    → [webform_submission:values:email], [site:name]
  ├── Email      → [user:name], [user:mail], [site:url]
  └── Rules      → tous les tokens disponibles

Accéder au Token Tree Browser :
  → Apparaît sous le champ de configuration comme "Browse available tokens"
  → OU /admin/help/token pour la liste complète
```

---

## Déboguer les Tokens

```bash
# Voir quels tokens sont disponibles pour node
drush php:eval "
\$info = \Drupal::service('token')->getInfo();
foreach (array_keys(\$info['tokens']['node'] ?? []) as \$token) {
  echo '[node:' . \$token . ']' . PHP_EOL;
}
" | head -30

# Tester un token avec une entité réelle
drush php:eval "
\$node = \Drupal::entityTypeManager()->getStorage('node')->load(42);
\$result = \Drupal::service('token')->replace(
  '[node:title] — [site:name]',
  ['node' => \$node],
  ['clear' => true]
);
echo \$result;
"
```
