---
name: drupal-seo — analytics et cookie consent
description: Intégrer analytics (Google Analytics GA4, Matomo) et cookie consent RGPD (Tarte au Citron, Cookiebot) sur les sites Drupal.
---

# Analytics & Cookie Consent RGPD — Référence Complète

## Choix du Module Analytics

```
Google Analytics 4 (GA4) :
  → drupal/google_analytics  — module dédié GA4, config via UI
  → drupal/google_tag        — Google Tag Manager (GTM), plus flexible

Matomo (auto-hébergé, RGPD natif) :
  → drupal/matomo            — module dédié, trackeur sans tiers
  → Avantage : données sur vos serveurs, RGPD sans banière si hébergé FR/EU

Contexte agence FR :
  → matomo : 8/20 projets   (OVH, Hetzner, infra FR)
  → google_analytics : 11/20 projets
  → google_tag (GTM) : 3/20 projets
```

---

## Google Analytics GA4

```bash
composer require drupal/google_analytics
drush en google_analytics -y
```

```yaml
# Configuration /admin/config/system/google-analytics
# Exporter vers config/default/google_analytics.settings.yml
account: G-XXXXXXXXXX          # ID GA4 (pas UA-)
visibility:
  request_path_mode: 0          # 0 = toutes les pages sauf liste
  request_path_pages: "/admin\n/admin/*"   # Pages exclues
codesnippet:
  before: ''
  after: ''
track:
  userid: true                  # Tracking User ID si utilisateur connecté
  mailto: false
  outbound: true                # Liens sortants
  files: true                   # Téléchargements fichiers
  files_extensions: 'pdf|doc|docx|xls|xlsx|zip|gz'
```

---

## Google Tag Manager (GTM) — drupal/google_tag

```bash
composer require drupal/google_tag
drush en google_tag -y
```

```php
// Surcharger les données du dataLayer depuis PHP
/**
 * Implements hook_google_tag_event_alter().
 */
function mon_module_google_tag_event_alter(array &$event, string $event_name): void {
  if ($event_name === 'page_view') {
    $node = \Drupal::routeMatch()->getParameter('node');
    if ($node instanceof \Drupal\node\Entity\Node) {
      $event['content_type'] = $node->bundle();
      $event['content_id'] = $node->id();
      // Catégorie du premier terme du champ tags
      if (!$node->get('field_tags')->isEmpty()) {
        $event['content_category'] = $node->get('field_tags')->entity?->getName();
      }
    }
  }
}
```

---

## Matomo — Analytics Auto-Hébergé

```bash
composer require drupal/matomo
drush en matomo -y
```

```yaml
# /admin/config/system/matomo → exporter en config YAML
# matomo.settings.yml
site_id: '1'
url_http: 'https://matomo.mon-domaine.fr/'
url_https: 'https://matomo.mon-domaine.fr/'
visibility:
  request_path_mode: 0
  request_path_pages: "/admin\n/admin/*"
tracking:
  userid_identification: true
  files_extensions: 'pdf|doc|docx|xls|xlsx|zip'
  outbound_links: true
  domain_mode: 0
codesnippet:
  before: ''
  after: ''
privacy:
  donottrack: true              # Respecter l'en-tête DNT
```

```php
// Ajouter des variables custom Matomo via hook
/**
 * Implements hook_matomo_custom_variables().
 */
function mon_module_matomo_custom_variables(): array {
  $variables = [];

  $node = \Drupal::routeMatch()->getParameter('node');
  if ($node) {
    $variables[1] = [
      'slot'  => 1,
      'name'  => 'Content Type',
      'value' => $node->bundle(),
      'scope' => 'page',  // page | visit | event
    ];
  }

  $user = \Drupal::currentUser();
  if (!$user->isAnonymous()) {
    $variables[2] = [
      'slot'  => 2,
      'name'  => 'User Role',
      'value' => implode(',', $user->getRoles(TRUE)),
      'scope' => 'visit',
    ];
  }

  return $variables;
}
```

---

## Tarte au Citron — Cookie Consent RGPD (FR standard)

Le module le plus utilisé en France pour la conformité RGPD des cookies.

```bash
composer require drupal/tarte_au_citron
drush en tarte_au_citron -y
```

```yaml
# /admin/config/system/tarte-au-citron
# tarte_au_citron.settings.yml
tac_services:
  - googletag          # Google Tag Manager
  - matomo             # Matomo/Piwik
  - googleanalytics    # Google Analytics
  - youtube            # YouTube iframes
  - vimeo              # Vimeo iframes
  - linkedin           # LinkedIn Insights
  - facebook           # Facebook Pixel
tac_locale: fr
tac_privacy_url: '/politique-confidentialite'
tac_hashtag: '#tarteaucitron'
tac_cookie_name: 'tarteaucitron'
tac_orientation: 'middle'        # bottom | middle | top
tac_show_alert_small: false
tac_group_services: false
tac_mandatory_services: []
```

```php
// Bloquer le script analytics jusqu'au consentement
// Tarte au citron gère cela automatiquement si les modules sont bien configurés
// Pour les scripts custom, utiliser la classe CSS "tac_visitor_optout"

// Dans un preprocess — conditionner le chargement d'une libraire au consentement
function mon_theme_preprocess_html(array &$variables): void {
  // Tarte au Citron gère le consentement côté JS
  // Ne pas charger les scripts analytics directement — laisser tac_* les gérer
}
```

```javascript
// JS custom pour interagir avec Tarte au Citron
// Vérifier le consentement depuis JavaScript
if (typeof tarteaucitron !== 'undefined' && tarteaucitron.state['googleanalytics']) {
  // GA accepté — pousser un événement
  gtag('event', 'custom_action', { 'event_category': 'UX' });
}

// Écouter l'événement de consentement
document.addEventListener('tac.consentUpdated', function(e) {
  console.log('Services acceptés:', e.detail);
});
```

---

## Cookiebot / CookieYes — Alternatives Internationales

```bash
# Cookiebot (IABTCF, pour les sites internationaux)
composer require drupal/cookiebot
drush en cookiebot -y

# Ou via module google_tag + config Consent Mode v2
```

---

## Anti-Patterns Analytics + RGPD

| ❌ À ne jamais faire | ✅ Bonne pratique | Impact |
|---------------------|------------------|--------|
| Charger GA sans consentement | Tarte au Citron ou Cookiebot obligatoire | Amende CNIL jusqu'à 4% du CA |
| ID GA4 en dur dans le template | Module drupal/google_analytics + config exportée | Non configurable par environnement |
| Matomo sur CDN externe (matomo.js) | Matomo auto-hébergé sur domaine FR/EU | Transfert de données hors EU |
| `gtag('config', ...)` sans consent mode | Google Consent Mode v2 + Tarte au Citron | Données UA non conformes |
| Tracking des utilisateurs connectés sans mention | `track.userid: false` par défaut, mention légale si activé | Violation RGPD |
| Analytics en dur dans settings.php | Config exportée en YAML, ID en variable d'env | ID prod en dev = données polluées |

---

## Vérifier la Conformité RGPD

```bash
# Tester que le script n'est pas chargé avant consentement
# Via DevTools Network, filtrer sur "analytics.js" ou "matomo.js"
# → Ne doit apparaître qu'APRÈS clic sur "Accepter"

# Test avec drush
drush php:eval "
\$config = \Drupal::config('matomo.settings');
echo 'Matomo URL: ' . \$config->get('url_https') . PHP_EOL;
echo 'Site ID: ' . \$config->get('site_id') . PHP_EOL;
"
```
