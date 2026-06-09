# Changelog — drupal-cron-queue

---

## v1.2 — 2026-06-09 (modernisation D11)

**Corrections :**
- `queue-workers.md` : attribut `#[QueueWorker]` désormais code principal (annotation `@QueueWorker` dépréciée D11) ; ajout des imports `Attribute\QueueWorker` + `TranslatableMarkup`
- `queue-workers.md` : couverture complète des exceptions queue — `RequeueException` (immédiat), `DelayedRequeueException` (backoff D9.3+, exemple 5xx), `SuspendQueueException`
- `cron-management.md` : nouvelle section **lock service** (`\Drupal::lock()`) pour cron long évitant les chevauchements
- `cron-management.md` + `SKILL.md` : Ultimate Cron marqué ⚠️ beta-only sur D11, recommandation crontab natif pour nouveaux projets
- `SKILL.md` : table d'évolution enrichie (RequeueException, DelayedRequeueException, attribut D10.2+) ; nouvelles entrées QDT + anti-pattern (lock, backoff)
- `lessons.md` : 2 leçons ajoutées (chevauchement cron sans lock, Ultimate Cron non-stable D11) ; leçon `cron.time` migrée vers la syntaxe attribut

---

## v1.1 — 2026-05-16 (audit complet)

**Corrections :**
- See Also mis à jour (drupal-tooling remplacé par drupal-deployment)
- Leçons enrichies (6 leçons au total)
- Fichiers manquants créés (liens QDT résolus)

---

## v1.0 — 2026-05-16

**Création initiale**

- SKILL.md avec Quick Decision Table (3 fichiers de référence)
- lessons.md avec incidents réels
