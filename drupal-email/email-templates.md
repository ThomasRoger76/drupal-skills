---
name: drupal-email — templates Twig
description: Écrire des templates Twig pour les emails Drupal avec Symfony Mailer - HTML responsive, CSS inline, version texte, et héritage de templates.
---

# Email Templates Twig — Référence Complète

## Structure des Templates

```
web/themes/custom/mon_theme/templates/email/

# Ou dans un module :
web/modules/custom/mon_module/templates/email/

Nom de template automatiquement découvert :
  mon_module--SOUS_TYPE.html.twig
  mon_module.html.twig (fallback)

# Exemple pour EmailBuilder 'mon_module' avec sub_type 'commande_confirmee' :
  mon_module--commande-confirmee.html.twig
```

---

## Template Base — `@symfony_mailer/base.html.twig`

```twig
{# Hériter du template de base Symfony Mailer #}
{% extends "@symfony_mailer/base.html.twig" %}

{# Surcharger le sujet #}
{% block subject %}
  Confirmation de votre commande {{ commande.reference.value }}
{% endblock %}

{# Corps HTML de l'email #}
{% block body %}
  {# Le CSS ici sera automatiquement inliné par Symfony Mailer #}
  <style>
    body { font-family: Arial, Helvetica, sans-serif; font-size: 16px; color: #333; }
    .header { background: #1a5276; padding: 20px; text-align: center; }
    .content { padding: 30px; max-width: 600px; margin: 0 auto; }
    .btn { display: inline-block; background: #1a5276; color: #fff !important;
           padding: 12px 24px; text-decoration: none; border-radius: 4px; }
    .footer { background: #f8f9fa; padding: 15px; text-align: center;
              font-size: 12px; color: #666; }
  </style>

  <div class="header">
    <img src="{{ absolute_url(base_path ~ directory ~ '/logo.svg') }}"
         alt="{{ site_name }}" height="50" style="display:block;margin:0 auto;">
  </div>

  <div class="content">
    <h1 style="color:#1a5276;margin-top:0;">Commande confirmée !</h1>

    <p>Bonjour <strong>{{ commande.get('uid').entity.displayname }}</strong>,</p>
    <p>Votre commande <strong>#{{ commande.reference.value }}</strong> a bien été reçue
       et est en cours de traitement.</p>

    <table style="width:100%;border-collapse:collapse;margin:20px 0;">
      <thead>
        <tr style="background:#f8f9fa;">
          <th style="padding:10px;text-align:left;border-bottom:2px solid #dee2e6;">Référence</th>
          <th style="padding:10px;text-align:right;border-bottom:2px solid #dee2e6;">Montant TTC</th>
        </tr>
      </thead>
      <tbody>
        <tr>
          <td style="padding:12px 10px;">{{ commande.reference.value }}</td>
          <td style="padding:12px 10px;text-align:right;">
            {{ commande.montant.value|number_format(2, ',', ' ') }} €
          </td>
        </tr>
      </tbody>
    </table>

    <p style="text-align:center;margin:30px 0;">
      <a href="{{ url('entity.commande.canonical', {commande: commande.id}) }}"
         class="btn" style="background:#1a5276;color:#fff !important;padding:12px 24px;
                             text-decoration:none;border-radius:4px;display:inline-block;">
        Voir ma commande
      </a>
    </p>

    <hr style="border:none;border-top:1px solid #dee2e6;margin:20px 0;">
    <p style="color:#666;font-size:14px;">
      Des questions ? Contactez-nous à
      <a href="mailto:{{ site_mail }}">{{ site_mail }}</a>
    </p>
  </div>

  <div class="footer">
    © {{ "now"|date("Y") }} {{ site_name }} —
    <a href="{{ site_url }}" style="color:#666;">{{ site_url }}</a><br>
    <a href="{{ url('user.unsubscribe') }}" style="color:#999;font-size:11px;">
      Se désabonner
    </a>
  </div>
{% endblock %}

{# Version texte (fallback pour clients email sans HTML) #}
{% block text %}
Confirmation de votre commande {{ commande.reference.value }}

Bonjour {{ commande.get('uid').entity.displayname }},

Votre commande #{{ commande.reference.value }} a bien été reçue.
Montant : {{ commande.montant.value|number_format(2, ',', ' ') }} €

Voir votre commande : {{ url('entity.commande.canonical', {commande: commande.id}) }}

Cordialement,
{{ site_name }}
{{ site_url }}
{% endblock %}
```

---

## Variables Automatiquement Disponibles

```twig
{# Variables injectées automatiquement par Symfony Mailer #}
{{ site_name }}          → Nom du site (depuis system.site.name)
{{ site_url }}           → URL du site
{{ site_mail }}          → Email admin
{{ base_path }}          → Chemin de base (/var/www/html/)
{{ directory }}          → Répertoire du thème courant

{# Variables injectées par setVariable() dans l'EmailBuilder #}
{{ commande }}           → L'entité Commande
{{ user }}               → L'entité User

{# Fonctions Twig disponibles #}
{{ url('route.name', {param: value}) }}   → URL Drupal absolue
{{ path('route.name', {param: value}) }}  → URL relative
{{ absolute_url(relative_url) }}          → Convertir en absolu
{{ "now"|date("Y") }}                     → Date courante formatée
```

---

## CSS Inline — Bonnes Pratiques

```twig
{# Symfony Mailer inline automatiquement le CSS dans des balises <style> #}
{# MAIS pour la compatibilité maximale — toujours doubler avec les attributs style= inline #}

{# ❌ Pas fiable dans tous les clients #}
<td class="important">texte</td>

{# ✅ Toujours fonctionnel (Gmail, Outlook, Apple Mail...) #}
<td style="font-weight:bold;color:#1a5276;">texte</td>

{# ✅ Les deux (recommandé — Symfony Mailer inline la classe ET tu as le fallback) #}
<td class="important" style="font-weight:bold;color:#1a5276;">texte</td>
```

---

## Multiples Templates par Module

```twig
{# Hiérarchie automatique de découverte : #}
{# mon_module--commande-confirmee.html.twig (le plus spécifique) #}
{# mon_module.html.twig (fallback pour tous les sub_types du module) #}

{# Template différent par langue : #}
{# mon_module--commande-confirmee.fr.html.twig #}
{# mon_module--commande-confirmee.en.html.twig #}
```

---

## Tester le Rendu du Template

```bash
# Voir le HTML généré (sans envoyer)
drush php:eval "
\$commande = \Drupal::entityTypeManager()->getStorage('commande')->load(1);
\$email = \Drupal::service('email_factory')
  ->newTypedEmail('mon_module', 'commande_confirmee', \$commande);
echo \$email->getHtmlBody();
" | head -50

# Preview dans le navigateur via l'interface Symfony Mailer
# /admin/config/system/mailer/policy/mon_module.commande_confirmee → Preview

# Tester dans Maildev
# docker compose up maildev -d → http://localhost:83
```
