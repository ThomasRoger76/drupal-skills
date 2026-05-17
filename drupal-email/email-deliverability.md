---
name: drupal-email — deliverability
description: Configurer SPF, DKIM, DMARC pour les emails Drupal - éviter les spams, configurer les DNS, tester la délivrabilité, et gérer les bounces.
---

# Email Deliverability — SPF, DKIM, DMARC

## Pourquoi la Délivrabilité est Critique

```
Problème courant : les emails Drupal arrivent en SPAM
  ❌ email envoyé depuis PHP sans auth SMTP → IP de l'hébergeur = IP non whitelistée
  ❌ Pas de SPF → les serveurs destinataires rejettent ou filtrent
  ❌ Pas de DKIM → pas de signature cryptographique = usurpation d'identité possible
  ❌ Domaine "From" ≠ domaine d'envoi → alignment DMARC fail

Solution : configurer SPF + DKIM + DMARC sur le domaine expéditeur
```

---

## SPF — Sender Policy Framework

SPF déclare quels serveurs sont autorisés à envoyer pour votre domaine.

```dns
# DNS Zone de mon-site.com — ajouter un enregistrement TXT
_         TXT "v=spf1 include:sendgrid.net include:mailgun.org ~all"

# Exemples selon le provider :
# SendGrid    : include:sendgrid.net
# Mailgun     : include:mailgun.org
# Amazon SES  : include:amazonses.com
# Gmail/Google: include:_spf.google.com
# OVH         : include:mx.ovh.com

# Syntaxe complète :
# v=spf1 [mechanisms] [qualifier]all
# ~all = softfail (marqué spam mais pas rejeté)
# -all = fail strict (rejeté)
# +all = dangereux (accepte tout)
# recommandé : ~all (softfail) pendant la mise en place, puis -all
```

```bash
# Vérifier votre SPF actuel
dig TXT mon-site.com | grep spf
# OU
nslookup -type=TXT mon-site.com | grep spf

# Tester avec un outil en ligne : mxtoolbox.com/spf.aspx
```

---

## DKIM — DomainKeys Identified Mail

DKIM ajoute une signature cryptographique à chaque email.

### Configuration SendGrid

```bash
# 1. Dans SendGrid Dashboard → Settings → Sender Authentication
# → Add Domain → entrer votre domaine
# SendGrid génère 3 enregistrements CNAME à ajouter dans votre DNS

# Enregistrements générés (exemple) :
s1._domainkey.mon-site.com   CNAME → s1.domainkey.u12345.wl001.sendgrid.net
s2._domainkey.mon-site.com   CNAME → s2.domainkey.u12345.wl001.sendgrid.net
em1234.mon-site.com          CNAME → u12345.wl001.sendgrid.net

# Vérifier après propagation DNS (24-48h)
dig CNAME s1._domainkey.mon-site.com
```

### Configuration Mailgun

```bash
# Dans Mailgun Dashboard → Sending → Domains → votre domaine
# → DNS Records → copier les enregistrements DKIM

# Exemple enregistrement DKIM Mailgun :
mailo._domainkey.mon-site.com  TXT  "k=rsa; p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA..."

# Vérifier :
dig TXT mailo._domainkey.mon-site.com | grep DKIM
```

### Configuration Amazon SES

```bash
# AWS Console → SES → Verified identities → votre domaine
# → View DKIM records → ajouter les 3 enregistrements CNAME

# OU avec AWS CLI :
aws ses verify-domain-dkim --domain mon-site.com
# → Retourne les tokens DKIM à ajouter en DNS
```

---

## DMARC — Politique de Conformité

DMARC indique aux serveurs destinataires quoi faire si SPF ou DKIM échoue.

```dns
# Enregistrement DMARC minimal (monitoring uniquement)
_dmarc.mon-site.com  TXT  "v=DMARC1; p=none; rua=mailto:dmarc-reports@mon-site.com"

# Progressif → passer à quarantine puis reject
# v=DMARC1  : version
# p=none     : ne rien faire (monitoring)
# p=quarantine : mettre en spam si fail
# p=reject   : rejeter si fail (le plus strict)
# rua=       : adresse de rapport agrégé (obligatoire pour monitoring)

# Exemple production :
_dmarc.mon-site.com  TXT  "v=DMARC1; p=quarantine; pct=100; rua=mailto:dmarc@mon-site.com; ruf=mailto:dmarc-ruf@mon-site.com"
```

---

## Tester la Délivrabilité

```bash
# Test 1 : envoyer un email de test vers mail-tester.com
# → https://www.mail-tester.com/ → envoyer email vers l'adresse fournie → voir le score

# Test 2 : MX Toolbox Email Health Check
# → https://mxtoolbox.com/emailhealth/ → entrer votre domaine

# Test 3 : depuis Drupal — envoyer vers un alias de test
drush php:eval "
\Drupal::service('email_factory')
  ->newEmail()
  ->setTo('test@mail-tester.com')  # ou votre propre alias
  ->setSubject('Test délivrabilité')
  ->setHtmlBody('<p>Test</p>')
  ->send();
"

# Test 4 : vérifier les headers de l'email reçu
# Gmail → ⋮ → Show original → vérifier "SPF: PASS", "DKIM: PASS", "DMARC: PASS"
```

---

## Gestion des Bounces

```php
// Configurer le Return-Path (adresse de rebond)
// settings.php
$config['symfony_mailer.settings']['default_from']['address'] = 'noreply@mon-site.com';

// Return-Path différent de From (pour capturer les bounces)
// Dans EmailBuilder::build()
public function build(EmailInterface $email): void {
  $email->setFrom('noreply@mon-site.com');
  $email->setReturnPath('bounces@mon-site.com');  // Adresse de gestion des bounces
}
```

```bash
# Surveiller les bounces avec SendGrid ou Mailgun
# → Dashboard → Suppressions → Bounces
# → Lister les adresses invalides et les supprimer de la DB Drupal

# Script Drush pour désactiver les users avec email bounced
drush php:eval "
\$bounced_emails = ['invalide@example.com'];  // Récupérer depuis l'API
foreach (\$bounced_emails as \$email) {
  \$users = \Drupal::entityTypeManager()->getStorage('user')
    ->loadByProperties(['mail' => \$email, 'status' => 1]);
  foreach (\$users as \$user) {
    \$user->block()->save();
    echo 'Bloqué: ' . \$email . PHP_EOL;
  }
}
"
```

---

## Checklist Délivrabilité Drupal

```
DNS :
[ ] Enregistrement SPF configuré et validé (mxtoolbox.com)
[ ] DKIM configuré chez le provider SMTP et DNS mis à jour
[ ] DMARC configuré en mode monitoring (p=none) → progresser vers reject
[ ] PTR (reverse DNS) configuré si envoi via VPS propre

Configuration :
[ ] Transport SMTP configuré (SendGrid, Mailgun, SES — pas PHP mail())
[ ] FROM address = domaine du site (pas gmail.com, pas localhost)
[ ] Return-Path configuré
[ ] List-Unsubscribe header ajouté si emails marketing

Tests :
[ ] Score ≥ 8/10 sur mail-tester.com
[ ] SPF: PASS / DKIM: PASS / DMARC: PASS dans les headers Gmail
[ ] Emails reçus en boîte principale (pas spam) sur Gmail + Outlook + Apple Mail
```
