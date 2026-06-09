---
name: drupal-webform — éléments
description: Tous les types d'éléments Webform Drupal - configuration, logique conditionnelle (#states), wizard multi-étapes, pré-remplissage, et RGPD.
---

# Webform Elements — Référence Complète

## Types d'Éléments Disponibles

```
/admin/structure/webform → Éditer un formulaire → Add element

Catégories :
├── Basic Elements
│   ├── textfield, textarea, email, tel, number, url
│   ├── checkbox, checkboxes, radios, select
│   ├── date, datetime, datelist
│   └── hidden, value (computed)
│
├── Advanced Elements
│   ├── managed_file (upload de fichier)
│   ├── signature (signature pad)
│   ├── rating (étoiles)
│   ├── range (slider)
│   ├── captcha
│   └── computed_twig (valeur calculée)
│
├── Composite Elements
│   ├── name (prénom + nom)
│   ├── address (adresse complète)
│   ├── contact (prénom + nom + email + téléphone)
│   └── custom composite
│
├── Markup Elements
│   ├── webform_markup (HTML statique)
│   ├── horizontal_rule
│   └── message
│
└── Containers
    ├── details (collapsible section)
    ├── fieldset (groupe de champs)
    └── wizard_page (étape wizard)
```

---

## Configuration d'un Élément (YAML)

```yaml
# Format YAML d'un formulaire Webform (éditable via UI ou YAML)
elements: |
  prenom:
    '#type': textfield
    '#title': 'Prénom'
    '#required': true
    '#maxlength': 100

  email:
    '#type': email
    '#title': 'Email'
    '#required': true

  message:
    '#type': textarea
    '#title': 'Message'
    '#required': true
    '#rows': 5
    '#maxlength': 2000

  fichier:
    '#type': managed_file
    '#title': 'Pièce jointe'
    '#required': false
    '#file_extensions': 'pdf doc docx'
    '#max_filesize': '5 MB'
    '#multiple': false

  newsletter:
    '#type': checkbox
    '#title': "Je souhaite recevoir la newsletter"
    '#default_value': false

  consentement_rgpd:
    '#type': checkbox
    '#title': "J'accepte que mes données soient utilisées pour traiter ma demande"
    '#required': true
    '#required_error': "Vous devez accepter les conditions pour soumettre ce formulaire."
```

---

## Logique Conditionnelle — `#states`

```yaml
# Afficher un champ uniquement si un autre champ a une valeur spécifique
telephone:
  '#type': tel
  '#title': 'Téléphone'
  '#states':
    visible:
      ':input[name="contact_methode"]':
        value: telephone    # Visible si contact_methode = "telephone"

# Rendre obligatoire si une checkbox est cochée
details_urgence:
  '#type': textarea
  '#title': 'Détails urgence'
  '#states':
    required:
      ':input[name="urgent"]':
        checked: true
    visible:
      ':input[name="urgent"]':
        checked: true

# Afficher si plusieurs conditions (AND)
champ_conditionnel:
  '#type': textfield
  '#states':
    visible:
      ':input[name="type"]':
        value: pro
      ':input[name="pays"]':
        value: france

# Afficher si l'une OU l'autre condition (OR)
champ_ou:
  '#type': textfield
  '#states':
    visible:
      - ':input[name="type"]':
          value: pro
      - ':input[name="type"]':
          value: association
```

---

## Formulaire Multi-Étapes (Wizard)

```yaml
# Activer le wizard : Settings → Wizard pages
# Puis ajouter des éléments de type 'webform_wizard_page'

elements: |
  infos_personnelles:
    '#type': webform_wizard_page
    '#title': 'Informations personnelles'
    prenom:
      '#type': textfield
      '#title': 'Prénom'
      '#required': true
    email:
      '#type': email
      '#title': 'Email'
      '#required': true

  details_demande:
    '#type': webform_wizard_page
    '#title': 'Votre demande'
    sujet:
      '#type': select
      '#title': 'Sujet'
      '#options':
        info: 'Demande d information'
        devis: 'Demande de devis'
        support: 'Support technique'
    message:
      '#type': textarea
      '#title': 'Message'
      '#required': true

  confirmation:
    '#type': webform_wizard_page
    '#title': 'Confirmation'
    '#description': 'Vérifiez vos informations avant de soumettre.'
    resume:
      '#type': webform_markup
      '#markup': '<p>Récapitulatif de votre demande :</p>'
```

---

## Pré-Remplissage des Champs

```yaml
# Depuis l'URL : ?prenom=Jean&email=jean@example.com

prenom:
  '#type': textfield
  '#title': 'Prénom'
  '#default_value': '[current-user:field_prenom]'  # depuis le profil utilisateur

email:
  '#type': email
  '#title': 'Email'
  '#default_value': '[current-user:mail]'  # email de l'utilisateur connecté

# Query string automatique :
# Si l'URL contient ?email=test@test.com
# ET que l'élément s'appelle "email"
# → le champ est pré-rempli automatiquement
```

```php
// Pré-remplir programmatiquement via hook
function mon_module_webform_submission_form_alter(array &$form, FormStateInterface $form_state, $form_id): void {
  // Récupérer le webform
  $webform = $form_state->getFormObject()->getWebform();

  if ($webform->id() !== 'mon_formulaire') {
    return;
  }

  // Pré-remplir un champ depuis la requête HTTP
  $request = \Drupal::request();
  $ref = $request->query->get('ref');

  if ($ref && isset($form['elements']['reference'])) {
    $form['elements']['reference']['#default_value'] = $ref;
  }
}
```

---

## Élément Computed (Valeur Calculée)

```yaml
# Calculer le prix total depuis des champs quantité × prix unitaire
total_ht:
  '#type': webform_computed_twig
  '#title': 'Total HT'
  '#mode': value
  '#template': '{{ (data.quantite * data.prix_unitaire)|number_format(2, ",", " ") }} €'
  '#states':
    visible:
      ':input[name="quantite"]':
        filled: true

# Concaténer prénom + nom
nom_complet:
  '#type': webform_computed_twig
  '#title': 'Nom complet'
  '#mode': value
  '#template': '{{ data.prenom }} {{ data.nom }}'
```

---

## CAPTCHA et Anti-Spam

```bash
# Honeypot : protection invisible, zéro friction utilisateur — à privilégier
composer require drupal/captcha drupal/recaptcha drupal/honeypot
docker compose exec php drush en captcha recaptcha honeypot -y
```

```yaml
# Honeypot (invisible — recommandé) — via settings du formulaire
# Settings → Spam protection → Enable honeypot

# CAPTCHA reCAPTCHA v2 (le module drupal/recaptcha fournit la v2 :
# "I'm not a robot" ou "Invisible reCAPTCHA badge")
captcha:
  '#type': captcha
  '#captcha_type': 'recaptcha/reCAPTCHA'
  '#captcha_admin_mode': false
```

> **reCAPTCHA v3 :** le module `drupal/recaptcha` ne gère que la v2. Pour la v3 (score sans
> interaction), installer `drupal/recaptcha_v3` qui ajoute un type CAPTCHA dédié, puis configurer le
> seuil de score dans `/admin/config/people/captcha/recaptcha_v3`.

---

## Élément Custom — `#[WebformElement]`

> **Config avant code.** 50+ éléments natifs + composites custom (créables via UI : Build → composite)
> couvrent la quasi-totalité des besoins. Ne créer un élément en PHP que pour un widget réellement
> spécifique (rendu, validation et masse de données qu'aucun composite ne sait produire).

En Drupal 11 / Webform 6.2+, les plugins d'élément utilisent l'attribut PHP `#[WebformElement]`
(`Drupal\webform\Attribute\WebformElement`) — l'annotation `@WebformElement` est dépréciée. Un élément
custom étend généralement une base existante (`WebformTextField`, `WebformElementBase`, etc.).

```php
<?php

declare(strict_types=1);

// src/Plugin/WebformElement/Siret.php
namespace Drupal\mon_module\Plugin\WebformElement;

use Drupal\Core\Form\FormStateInterface;
use Drupal\Core\StringTranslation\TranslatableMarkup;
use Drupal\webform\Attribute\WebformElement;
use Drupal\webform\Plugin\WebformElement\TextField;
use Drupal\webform\WebformSubmissionInterface;

/**
 * Champ SIRET (numéro d'établissement français, 14 chiffres, clé de Luhn).
 */
#[WebformElement(
  id: 'siret',
  label: new TranslatableMarkup('SIRET'),
  description: new TranslatableMarkup('Numéro SIRET français validé (14 chiffres + Luhn).'),
  category: new TranslatableMarkup('Mon Module'),
)]
class Siret extends TextField {

  /**
   * {@inheritdoc}
   */
  protected function defineDefaultProperties(): array {
    return [
      '#maxlength' => 14,
      '#pattern' => '\d{14}',
    ] + parent::defineDefaultProperties();
  }

  /**
   * {@inheritdoc}
   *
   * Validation serveur — ne jamais se fier au seul #pattern (bypassable côté client).
   */
  public function prepare(array &$element, ?WebformSubmissionInterface $webform_submission = NULL): void {
    parent::prepare($element, $webform_submission);
    // Validation Luhn côté serveur, callback ajouté au pipeline de l'élément.
    $element['#element_validate'][] = [static::class, 'validateSiret'];
  }

  /**
   * Valide la clé de Luhn du SIRET.
   */
  public static function validateSiret(array &$element, FormStateInterface $form_state): void {
    $value = $form_state->getValue($element['#parents']);
    if ($value === '' || $value === NULL) {
      return;
    }
    if (!self::isLuhnValid($value)) {
      $form_state->setError($element, new TranslatableMarkup('Le numéro SIRET est invalide.'));
    }
  }

  /**
   * Algorithme de Luhn.
   */
  protected static function isLuhnValid(string $number): bool {
    $sum = 0;
    $alt = FALSE;
    for ($i = strlen($number) - 1; $i >= 0; $i--) {
      $digit = (int) $number[$i];
      if ($alt) {
        $digit *= 2;
        if ($digit > 9) {
          $digit -= 9;
        }
      }
      $sum += $digit;
      $alt = !$alt;
    }
    return $sum % 10 === 0;
  }
}
```

Après ajout : `docker compose exec php drush cr` puis l'élément `siret` apparaît dans Build → Add element.
