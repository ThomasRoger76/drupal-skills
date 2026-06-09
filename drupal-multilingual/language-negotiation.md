---
name: drupal-multilingual — language negotiation
description: Configuration de la négociation de langue Drupal - URL prefix, domain, session, browser detection, et API PHP pour détecter et changer la langue.
---

# Language Negotiation — Référence Complète

## Les Méthodes de Négociation (par priorité)

```
/admin/config/regional/language/detection

Méthodes disponibles (ordre = priorité) :
1. URL (prefix ou domain)    ← Recommandé en premier
2. Session                   ← Persiste via $_SESSION
3. User preference           ← Préférence du compte utilisateur
4. Browser                   ← Accept-Language header HTTP
5. Interface default         ← Langue par défaut du site (toujours en dernier)
```

---

## Mode URL Prefix

```
/fr/mon-article     → langue FR
/en/my-article      → langue EN
/de/mein-artikel    → langue DE
/mon-article        → langue par défaut (pas de préfixe)
```

**Configuration :** deux fichiers core distincts — `language.types.yml` (méthodes +
poids) et `language.negotiation.url.yml` (préfixes). Le fichier
`language.negotiation.yml` n'existe pas dans Drupal core.

```yaml
# config/sync/language.types.yml — méthodes activées + poids
negotiation:
  language_interface:
    enabled:
      language-url: -8         # Priorité la plus haute (négatif = premier)
      language-session: -4
      language-browser: -2
      language-selected: 12
    method_weights:
      language-url: -8
      language-session: -4
      language-browser: -2
      language-selected: 12
```

```yaml
# config/sync/language.negotiation.url.yml — préfixes URL
source: path_prefix
prefixes:
  fr: fr
  en: en
  de: de
  # La langue par défaut peut avoir un préfixe vide (pas de /fr/)
```

**Langue par défaut sans préfixe :**
```
/admin/config/regional/language/detection/url

→ "Path prefix configuration" :
  - fr → fr
  - en → "" (vide = pas de préfixe pour l'anglais)
  - de → de

→ URLs :
  /mon-article        → anglais (par défaut, pas de préfixe)
  /fr/mon-article     → français
  /de/mein-artikel    → allemand
```

---

## Mode Domain

```
fr.mon-site.com     → langue FR
en.mon-site.com     → langue EN
de.mon-site.com     → langue DE
```

**Configuration :**
```yaml
# config/sync/language.negotiation.url.yml
source: domain
prefixes: {}
domains:
  fr: fr.mon-site.com
  en: en.mon-site.com
  de: de.mon-site.com
```

**Prérequis :** DNS + certificats SSL par sous-domaine.

---

## API PHP — Détection et Manipulation

```php
// Obtenir la langue courante
$language = \Drupal::languageManager()->getCurrentLanguage();
$langcode = $language->getId();  // 'fr', 'en', 'de'
$name = $language->getName();    // 'French', 'English'
$direction = $language->getDirection();  // 'ltr' ou 'rtl'

// Obtenir la langue courante pour le contenu
$content_language = \Drupal::languageManager()
  ->getCurrentLanguage(\Drupal\Core\Language\LanguageInterface::TYPE_CONTENT);

// Obtenir la langue de l'interface
$interface_language = \Drupal::languageManager()
  ->getCurrentLanguage(\Drupal\Core\Language\LanguageInterface::TYPE_INTERFACE);

// Lister toutes les langues actives
$languages = \Drupal::languageManager()->getLanguages();
foreach ($languages as $langcode => $language) {
  echo $langcode . ': ' . $language->getName() . PHP_EOL;
}

// Obtenir une langue spécifique
$fr = \Drupal::languageManager()->getLanguage('fr');

// Langue par défaut du site
$default = \Drupal::languageManager()->getDefaultLanguage();

// Vérifier si une langue est la défaut
$is_default = $language->isDefault();
```

---

## Générer des URLs par Langue

```php
use Drupal\Core\Url;
use Drupal\language\Entity\ConfigurableLanguage;

// URL d'une route dans une langue spécifique
$language_fr = ConfigurableLanguage::load('fr');
$url = Url::fromRoute('entity.node.canonical', ['node' => 42], [
  'language' => $language_fr,
  'absolute' => TRUE,
]);
$url_string = $url->toString();
// → https://mon-site.com/fr/mon-article (avec prefix URL)
// → https://fr.mon-site.com/mon-article (avec domain mode)

// URL de la page courante dans une autre langue
$request = \Drupal::request();
$current_url = \Drupal\Core\Url::createFromRequest($request);
$en_url = $current_url->setOption('language', ConfigurableLanguage::load('en'));

// Toutes les URLs d'une entité (pour le language switcher)
$node = Node::load(42);
$languages = \Drupal::languageManager()->getLanguages();
foreach ($languages as $langcode => $language) {
  if ($node->hasTranslation($langcode)) {
    $url = $node->getTranslation($langcode)->toUrl('canonical', [
      'language' => $language,
      'absolute' => TRUE,
    ]);
    echo $langcode . ': ' . $url->toString() . PHP_EOL;
  }
}
```

---

## Language Switcher Block Custom

```php
<?php
// src/Plugin/Block/LanguageSwitcherBlock.php
namespace Drupal\mon_module\Plugin\Block;

use Drupal\Core\Block\BlockBase;
use Drupal\language\Entity\ConfigurableLanguage;

/**
 * @Block(
 *   id = "mon_module_language_switcher",
 *   admin_label = @Translation("Language Switcher Custom"),
 * )
 */
class LanguageSwitcherBlock extends BlockBase {

  public function build(): array {
    $languages = \Drupal::languageManager()->getLanguages();
    $current_langcode = \Drupal::languageManager()->getCurrentLanguage()->getId();
    $path_processor = \Drupal::service('path_processor_manager');
    $request = \Drupal::request();

    $links = [];
    foreach ($languages as $langcode => $language) {
      $url = \Drupal\Core\Url::createFromRequest($request);
      $url->setOption('language', $language);

      $links[$langcode] = [
        'title' => $language->getName(),
        'url' => $url,
        'language' => $language,
        'active' => $langcode === $current_langcode,
      ];
    }

    return [
      '#theme' => 'language_switcher_custom',
      '#links' => $links,
      '#current_language' => $current_langcode,
      '#cache' => [
        'contexts' => ['languages:language_interface', 'url'],
      ],
    ];
  }
}
```

---

## Détecter la Langue depuis le Browser

```php
// Lire l'header Accept-Language et sélectionner la meilleure langue
$request = \Drupal::request();
$accept_language = $request->headers->get('Accept-Language', '');
// → "fr-FR,fr;q=0.9,en-US;q=0.8,en;q=0.7"

// Drupal fait ça automatiquement si la méthode 'browser' est activée
// dans la negotiation — mais voici comment le faire manuellement :
$negotiator = \Drupal::service('language_negotiator');
$method = \Drupal::service('plugin.manager.language_negotiation_method')
  ->createInstance('language-browser');
$detected = $method->getLangcode($request);
// → 'fr' si Accept-Language contient fr

// Dans les settings : le middleware Drupal lit automatiquement
// le header et applique la méthode 'browser' si configurée
```

---

## Troubleshooting

```bash
# Vérifier quelle méthode est utilisée pour détecter la langue
docker compose exec php drush php:eval "
\$negotiator = \Drupal::service('language_negotiator');
\$methods = \$negotiator->getNegotiationMethods();
foreach (\$methods as \$id => \$method) {
  echo \$id . ': ' . \$method['name'] . PHP_EOL;
}
"

# Vérifier les prefixes configurés
docker compose exec php drush config:get language.negotiation.url prefixes

# Tester la détection depuis un header spécifique
curl -H "Accept-Language: fr-FR,fr;q=0.9" https://mon-site.com/
# → Drupal doit retourner la version FR si 'browser' est configuré
```
