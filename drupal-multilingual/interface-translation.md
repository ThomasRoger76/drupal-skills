---
name: drupal-multilingual — interface translation
description: Traduction de l'interface Drupal (strings PHP, modules, thèmes) avec le module Interface Translation - fichiers .po, drush locale, YAML, et gestion des traductions custom.
---

# Interface Translation — Référence Complète

## Qu'est-ce que l'Interface Translation

Le module Interface Translation (aussi appelé `locale`) traduit les **chaînes PHP** de l'interface Drupal :
- Textes dans le code PHP : `t('Save')`, `$this->t('Save')`
- Labels dans les fichiers `.install`, `.module`
- Chaînes dans les templates Twig : `{{ 'Save'|t }}`
- Labels dans les fichiers de config YAML (définitions de champs, etc.)

---

## Traduire une Chaîne PHP

```php
// Dans un module — toujours utiliser $this->t() ou t()
class MonService {
  public function getMessage(): string {
    // Traduction simple
    return $this->t('Commande confirmée.');

    // Avec variables
    return $this->t('Commande @ref confirmée.', ['@ref' => $reference]);

    // Avec contexte (disambiguïté)
    return $this->t('Post', [], ['context' => 'Verb: to post something']);
    // vs
    return $this->t('Post', [], ['context' => 'Noun: a post/article']);

    // Pour un utilisateur spécifique (langue de l'utilisateur, pas courante)
    return new TranslatableMarkup('Message', [], ['langcode' => $user->getPreferredLangcode()]);
  }
}

// Dans les fichiers .install
function mon_module_install(): void {
  \Drupal::messenger()->addStatus(
    t('Module Mon Module installé avec succès.')
  );
}
```

```twig
{# Dans les templates Twig #}
{{ 'Save'|t }}
{{ 'Hello @name'|t({'@name': user.name}) }}
{% trans %}
  Hello {{ name }}!
{% endtrans %}
{% trans with {'context': 'greeting'} %}
  Hello {{ name }}!
{% endtrans %}
```

---

## Fichiers .po — Format et Import

### Format d'un fichier .po

```po
# Fichier de traduction française pour mon_module
# Copyright 2026 Mon Projet
msgid ""
msgstr ""
"Project-Id-Version: mon_module\n"
"Language: fr\n"
"Content-Type: text/plain; charset=UTF-8\n"
"Content-Transfer-Encoding: 8bit\n"

msgid "Save"
msgstr "Enregistrer"

msgid "Delete"
msgstr "Supprimer"

# Chaîne avec variable
msgid "Order @ref confirmed."
msgstr "Commande @ref confirmée."

# Chaîne plurielle
msgid "@count item"
msgid_plural "@count items"
msgstr[0] "@count élément"
msgstr[1] "@count éléments"

# Chaîne avec contexte
msgctxt "Verb: to post something"
msgid "Post"
msgstr "Publier"

msgctxt "Noun: a post/article"
msgid "Post"
msgstr "Article"
```

### Importer via Drush

```bash
# Importer un fichier .po
drush locale:import fr /chemin/vers/fr.po

# Forcer l'override des traductions existantes
drush locale:import fr /chemin/vers/fr.po --override=all

# Importer pour une langue spécifique avec type
drush locale:import fr /chemin/vers/fr.po --type=customized

# Types :
# customized = traductions personnalisées (ne seront pas écrasées par locale:update)
# not-customized = traductions contrib (peuvent être mises à jour)
```

---

## Mettre à Jour les Traductions Contrib

```bash
# Télécharger et mettre à jour les traductions depuis drupal.org
drush locale:update

# Vérifier l'état des traductions
drush locale:check

# Statistiques de traduction
drush php:eval "
\$stats = locale_translation_get_file_history();
foreach (\$stats as \$module => \$info) {
  if (isset(\$info['fr'])) {
    echo \$module . ': ' . (\$info['fr']->last_checked ? date('Y-m-d', \$info['fr']->last_checked) : 'jamais') . PHP_EOL;
  }
}
"
```

---

## Exporter les Traductions Custom

```bash
# Exporter uniquement les traductions personnalisées (customized)
drush locale:export fr --types=customized > fr-custom.po

# Exporter toutes les traductions d'une langue
drush locale:export fr > fr-all.po
```

---

## Traduction dans les Fichiers de Config YAML

```yaml
# Les labels dans les YAML de config sont automatiquement extraits
# si wrappés correctement dans le code PHP

# config/install/field.field.node.article.field_resume.yml
label: 'Résumé'
description: 'Résumé court de l''article.'

# Ces chaînes peuvent être traduites via :
# /admin/config/regional/config-translation
```

---

## Programmation — Forcer une Langue

```php
// Obtenir une traduction dans une langue spécifique (pas la langue courante)
$translation = \Drupal::translation()->translate(
  'Order @ref confirmed.',
  ['@ref' => 'CMD-001'],
  ['langcode' => 'fr']
);

// t() avec langcode explicite
$message = (string) t('Save', [], ['langcode' => 'de']);

// TranslationInterface::formatPlural()
$count = 5;
$message = \Drupal::translation()->formatPlural(
  $count,
  '@count item',
  '@count items',
  ['@count' => $count],
  ['langcode' => 'fr']
);
// → "5 éléments"
```

---

## Vérifier et Debugger

```bash
# Voir les chaînes non traduites pour une langue
drush php:eval "
\$query = \Drupal::database()->select('locales_source', 'ls');
\$query->leftJoin('locales_target', 'lt', 'ls.lid = lt.lid AND lt.language = :lang', [':lang' => 'fr']);
\$query->fields('ls', ['source', 'context']);
\$query->isNull('lt.lid');
\$query->range(0, 20);
\$results = \$query->execute()->fetchAll();
foreach (\$results as \$r) {
  echo \$r->source . PHP_EOL;
}
"

# Nombre de chaînes traduites vs total
drush php:eval "
\$stats = locale_translation_status();
if (isset(\$stats['drupal']['fr'])) {
  \$s = \$stats['drupal']['fr'];
  echo 'Traduites: ' . (\$s->last_checked ?? 0) . PHP_EOL;
}
"

# Nettoyer le cache des traductions
drush locale:check
drush cr
```
