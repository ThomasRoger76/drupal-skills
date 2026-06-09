# Leçons — drupal-performance

Incidents de performance résolus en production. Mis à jour après chaque résolution.

---

## 2026-05-16 — Création du skill

### `drush cr` n'améliore pas les performances — cause non identifiée
- **Symptôme :** Page lente, `drush cr` lancé en boucle sans effet
- **Cause :** Le vrai problème est ailleurs — une requête SQL lente, un block sans cache tags, un appel API externe bloquant
- **Correct :** Utiliser le slow query log + Devel query log pour identifier la vraie cause avant d'agir
- **Prévention :** `drush cr` est un outil de debug, pas une solution de performance. Diagnostiquer d'abord.

### Render array sans `#cache` → contenu périmé affiché
- **Symptôme :** Après modification d'un nœud, l'ancienne version s'affiche pendant des heures
- **Cause :** Le render array du bloc n'avait pas `#cache['tags'] => $node->getCacheTags()` → pas d'invalidation lors du save()
- **Correct :** Ajouter `'tags' => $node->getCacheTags()` sur tous les render arrays qui dépendent d'une entité
- **Prévention :** Règle : tout render array qui affiche des données d'une entité DOIT avoir ses cache tags

### `max-age: 0` global → Dynamic Page Cache désactivé
- **Symptôme :** Toutes les pages pour utilisateurs connectés sont lentes — Dynamic Page Cache inactif
- **Cause :** Un bloc avec `'max-age' => 0` sans cache contexts fait que Drupal désactive DPC sur toute la page
- **Correct :** `'max-age' => 0` uniquement sur le composant vraiment dynamique, avec des cache contexts corrects
- **Prévention :** `max-age: 0` doit être accompagné de contexts précis — sinon toute la page est exclue du cache

### N+1 queries sur entity references dans preprocess
- **Symptôme :** 200 requêtes SQL pour une liste de 20 nœuds avec champs Entity Reference
- **Cause :** `$node->get('field_auteur')->entity` dans un foreach → 1 requête par nœud
- **Correct :** Collecter tous les IDs → `User::loadMultiple($ids)` → map par UID
- **Prévention :** Jamais de `->entity` dans un foreach sur des résultats Views — utiliser preRender() avec batch loading

### Varnish cache jamais invalidé après modification de nœud
- **Symptôme :** Modifications de contenu non visibles pendant des heures sur le site en production
- **Cause :** Le module Purge n'était pas configuré — Varnish n'était pas notifié lors des saves Drupal
- **Correct :** Installer `drupal/purge` + `drupal/varnish_purger` → configurer les purgers avec les cache tags HTTP
- **Prévention :** Toute stack Varnish + Drupal doit avoir Purge configuré. Vérifier avec `curl -I url | grep X-Drupal-Cache-Tags`

### OPcache `validate_timestamps = 1` en production
- **Symptôme :** Performances PHP anormalement basses malgré OPcache actif
- **Cause :** `validate_timestamps = 1` force PHP à vérifier la date de modification de chaque fichier à chaque requête
- **Correct :** `validate_timestamps = 0` en production + `drush cr` après chaque déploiement pour invalider OPcache
- **Prévention :** Template `php-prod.ini` avec `validate_timestamps = 0` et `max_accelerated_files = 20000`

### BigPipe désactivé — blocs personnalisés bloquent tout le rendu
- **Symptôme :** TTFB (Time To First Byte) de 3-5 secondes pour les utilisateurs connectés
- **Cause :** BigPipe était désactivé dans la config — tous les blocs personnalisés bloquaient le rendu de la page
- **Correct :** `drush pm:enable big_pipe -y` + vérifier que les blocs utilisent `#lazy_builder` pour le contenu dynamique
- **Prévention :** BigPipe doit être actif sur tout site avec des blocs personnalisés. Vérifier `/admin/reports/status`

### Redis — connexion échoue silencieusement → retour sur database backend
- **Symptôme :** Site lent malgré Redis "configuré" — cache_default utilise la DB
- **Cause :** Redis n'était pas accessible (port fermé) → Drupal fallback sur database sans erreur visible
- **Correct :** Tester la connexion Redis : `drush php:eval "echo \Drupal::cache()->get('test') ? 'OK' : 'MISS';"` + vérifier les logs
- **Prévention :** Ajouter un health check Redis dans le Makefile d'installation. Monitorer `redis-cli info stats | grep keyspace`

---

## 2026-06-09 — Corrections techniques (audit qualité)

### `opcache.preload` qui boucle sur tout `vendor/` → PHP-FPM ne démarre plus
- **Symptôme :** Après activation du preload, fatal errors au boot PHP-FPM (`Cannot declare class … already in use`, classes non résolvables)
- **Cause :** Compiler récursivement tout `vendor/` ignore l'ordre des dépendances et le code conditionnel. Drupal ne supporte pas officiellement le preload (issue #3055735)
- **Correct :** Ne PAS activer le preload par défaut. Si nécessaire : liste curée de classes stables + `try/catch` autour de `opcache_compile_file()`
- **Prévention :** OPcache classique + APCu suffisent sur la majorité des projets. Tester le redémarrage PHP-FPM après tout ajout au preload

### `getenv('APP_ENV', 'prod')` → la valeur par défaut est ignorée silencieusement
- **Symptôme :** Préfixe de cache vide en l'absence de la variable d'env, collisions de clés entre environnements
- **Cause :** `getenv()` de PHP ne prend pas de 2e argument de valeur par défaut (à la différence de la fonction `env()` Symfony/Laravel)
- **Correct :** `(getenv('APP_ENV') ?: 'prod')` avec l'opérateur Elvis
- **Prévention :** En settings.php Drupal, toujours encadrer `getenv()` par `?:` pour la valeur de repli

### Context `user` confondu avec une exclusion du Dynamic Page Cache
- **Symptôme :** Croyance que `'contexts' => ['user']` empêche la mise en cache DPC
- **Cause :** Le DPC met bien en cache une variante par utilisateur ; c'est `'max-age' => 0` qui exclut, pas le context
- **Correct :** Pour partager le cache, préférer `user.roles`. Réserver `max-age 0` aux composants réellement non cacheables
- **Prévention :** Distinguer « varier le cache » (context) de « ne pas cacher » (max-age 0)
