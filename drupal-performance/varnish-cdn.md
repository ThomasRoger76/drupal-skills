---
name: drupal-performance — varnish et CDN
description: Configuration de Varnish avec le module Purge pour Drupal, VCL Drupal, invalidation par cache tags HTTP, CDN integration, et gestion des headers de cache.
---

# Varnish & CDN pour Drupal — Référence Complète

## Architecture Varnish + Drupal

```
Client → Varnish (port 80/443) → Drupal (port 8080 interne)

Varnish cache :
├── Pages anonymes → réponse complète en mémoire
├── Cache invalide via X-Drupal-Cache-Tags → purge sélective
└── Pass → requêtes avec cookies/authentification (utilisateurs connectés)
```

---

## Service Varnish dans Docker Compose

```yaml
# docker-compose.yml
services:
  varnish:
    image: varnish:7.4
    ports:
      - "80:80"
    volumes:
      - ./services/varnish/drupal.vcl:/etc/varnish/default.vcl
    environment:
      VARNISH_SIZE: 256M
    depends_on:
      - php

  php:
    # Pas de port 80 exposé directement — Varnish est le point d'entrée
    expose:
      - "80"
```

---

## VCL Drupal — Configuration Complète

```vcl
# services/varnish/drupal.vcl
vcl 4.1;

import std;
import purge;

# Backend Drupal (PHP/Apache)
backend default {
  .host = "php";
  .port = "80";
  .connect_timeout = 5s;
  .first_byte_timeout = 60s;
  .between_bytes_timeout = 60s;
}

# ACL pour les purges — autoriser uniquement Drupal lui-même
acl purge_allowed {
  "localhost";
  "php";           # Nom du service Docker
  "127.0.0.1";
}

# ─── vcl_recv : Traitement des requêtes entrantes ──────────────────────────

sub vcl_recv {
  # Toujours passer les requêtes PURGE au handler spécial
  if (req.method == "PURGE") {
    if (!client.ip ~ purge_allowed) {
      return (synth(403, "Access denied."));
    }
    return (purge);
  }

  # BAN par tags (utilisé par le module Purge de Drupal)
  if (req.method == "BAN") {
    if (!client.ip ~ purge_allowed) {
      return (synth(403, "Access denied."));
    }
    ban("obj.http.X-Drupal-Cache-Tags ~ " + req.http.Cache-Tags);
    return (synth(200, "Ban added"));
  }

  # Ne pas cacher les requêtes avec cookie de session (utilisateurs connectés)
  if (req.http.Cookie ~ "SESS") {
    return (pass);   # → Drupal directement, pas de cache Varnish
  }

  # Ne pas cacher les pages admin
  if (req.url ~ "^/admin" || req.url ~ "^/user") {
    return (pass);
  }

  # Supprimer les cookies non-essentiels (tracking, analytics)
  set req.http.Cookie = regsuball(req.http.Cookie, "(__utm|_ga|_gid|_gat)[^;]*;? ?", "");
  set req.http.Cookie = regsub(req.http.Cookie, "^;\s*", "");
  if (req.http.Cookie == "") {
    unset req.http.Cookie;
  }

  # Normaliser l'URL (supprimer les trailing slash doubles)
  set req.url = regsuball(req.url, "//+", "/");

  return (hash);
}

# ─── vcl_backend_response : Traitement des réponses de Drupal ─────────────

sub vcl_backend_response {
  # Durée de cache par défaut pour les pages Drupal anonymes
  if (beresp.http.Cache-Control ~ "public" && beresp.status == 200) {
    set beresp.ttl = 1h;
    set beresp.grace = 6h;   # Servir le contenu périmé si Drupal est lent
  }

  # Garder les X-Drupal-Cache-Tags dans le cache Varnish (pour les BAN)
  if (beresp.http.X-Drupal-Cache-Tags) {
    set beresp.http.X-Drupal-Cache-Tags-Store = beresp.http.X-Drupal-Cache-Tags;
  }

  # Ne pas cacher les erreurs
  if (beresp.status >= 400) {
    set beresp.ttl = 0s;
    set beresp.uncacheable = true;
  }

  return (deliver);
}

# ─── vcl_deliver : Traitement des réponses envoyées au client ─────────────

sub vcl_deliver {
  # Restaurer les cache tags depuis le cache pour les BAN futurs
  if (obj.http.X-Drupal-Cache-Tags-Store) {
    set resp.http.X-Drupal-Cache-Tags = obj.http.X-Drupal-Cache-Tags-Store;
  }

  # Header debug (HIT ou MISS)
  if (obj.hits > 0) {
    set resp.http.X-Cache = "HIT";
    set resp.http.X-Cache-Hits = obj.hits;
  } else {
    set resp.http.X-Cache = "MISS";
  }

  # Supprimer les headers Drupal internes en production
  # (commenter en dev pour le debug)
  # unset resp.http.X-Generator;
  # unset resp.http.X-Drupal-Cache;

  return (deliver);
}
```

---

## Module Purge — Invalidation depuis Drupal

```bash
# Installation
composer require drupal/purge drupal/varnish_purger
drush en purge purge_queuer_coretags purge_processor_cron varnish_purger -y
```

### Configuration du Purger

```yaml
# config/install/varnish_purger.settings.yml
id: varnish
name: 'Varnish Purger'
plugin: varnishpurger
runtime_measurement: true
cooldown_time: '0.2'
max_requests: '100'
request_method: BAN     # ← BAN est plus efficace que PURGE sur les tags
scheme: http
host: varnish            # Nom du service Docker Varnish
port: '80'
path: /
body_method: ban.body
body: ''

# Header envoyé lors d'un BAN :
# Cache-Tags: node:42 node_list
# Varnish applique le BAN sur les objets dont X-Drupal-Cache-Tags contient ces valeurs
```

### Configuration du Queuer

```yaml
# config/install/purge.plugins.yml
purgers:
  varnish:
    plugin_id: varnishpurger

queuers:
  coretags:
    plugin_id: coretags   # Écoute les invalidations de cache Drupal

processors:
  cron:
    plugin_id: cron       # Traite la queue lors du cron
  lateruntime:
    plugin_id: lateruntime  # Traite immédiatement (risque de ralentissement)
```

---

## CDN — Intégration

### Module CDN (contrib)

```bash
composer require drupal/cdn
drush en cdn -y
```

```yaml
# config/install/cdn.settings.yml — variante 1 : un seul domaine CDN
status: true
mapping:
  type: simple
  domain: cdn.mon-site.com
farfuture:
  status: false   # Far Future headers pour les assets statiques
```

```yaml
# config/install/cdn.settings.yml — variante 2 : plusieurs domaines (sharding)
# ⚠️ Ne PAS mélanger 'type: simple' et 'type: auto-balanced' dans le même mapping.
status: true
mapping:
  type: auto-balanced
  domains:
    - static1.mon-site.com
    - static2.mon-site.com
farfuture:
  status: false
```

### CDN Manuel (sans module)

```php
// Dans hook_file_url_alter() — modifier les URLs des fichiers
function mon_module_file_url_alter(&$uri): void {
  // Rediriger les fichiers public:// vers le CDN
  if (str_starts_with($uri, 'public://')) {
    $cdn_base = \Drupal::config('mon_module.settings')->get('cdn_url');
    if ($cdn_base) {
      $uri = str_replace(
        \Drupal::service('stream_wrapper_manager')->getViaScheme('public')->getExternalUrl(),
        $cdn_base . '/sites/default/files/',
        $uri
      );
    }
  }
}
```

---

## Headers de Cache Drupal pour Varnish

```php
// Configurer Drupal pour envoyer les bons headers au reverse proxy
// settings.php
$settings['reverse_proxy'] = TRUE;
$settings['reverse_proxy_addresses'] = [
  getenv('VARNISH_IP') ?: '172.18.0.0/16',  // IPs Varnish/Docker network
];
$settings['reverse_proxy_trusted_headers'] = \Symfony\Component\HttpFoundation\Request::HEADER_X_FORWARDED_FOR |
  \Symfony\Component\HttpFoundation\Request::HEADER_X_FORWARDED_HOST |
  \Symfony\Component\HttpFoundation\Request::HEADER_X_FORWARDED_PORT |
  \Symfony\Component\HttpFoundation\Request::HEADER_X_FORWARDED_PROTO;
```

---

## Vérification et Debug

```bash
# Vérifier les headers de cache d'une page
curl -I http://mon-site.com/node/42
# X-Cache: MISS  ← premier accès
# X-Drupal-Cache-Tags: node:42 node_list user:0
# Cache-Control: max-age=3600, public

curl -I http://mon-site.com/node/42
# X-Cache: HIT  ← depuis Varnish

# Tester la purge manuellement
curl -X BAN http://varnish:80/ \
  -H "Cache-Tags: node:42" \
  -H "X-Purge-Method: regex"

# Stats Varnish
docker compose exec varnish varnishstat -1 | grep -E "cache_hit|cache_miss|hit_rate"

# Logs Varnish en temps réel
docker compose exec varnish varnishlog -i ReqURL | head -50
```
