# Paramètres administrables — moteur de règles métier

Inventaire des paramètres qu'un administrateur fonctionnel doit pouvoir éditer depuis Odoo **sans modification de code**. Référencé par les capacités CAP-7 à CAP-14 du SPEC.

## Activation et déclencheur

| Paramètre | Type | Rôle |
|---|---|---|
| `suspension_enabled` | Boolean | Active ou désactive les suspensions automatiques globalement |
| `overdue_invoice_status` | Selection | Statut Odoo de facture qui déclenche le workflow (par défaut `late`) |

## Délais et grâce

| Paramètre | Type | Rôle |
|---|---|---|
| `suspension_delay_value` | Integer | Valeur du délai avant suspension après déclenchement |
| `suspension_delay_unit` | Selection | Unité du délai : `minute` / `heure` / `jour` |
| `allow_grace_period` | Boolean | Autorise l'application d'un délai de grâce |
| Nombre jours grâce | Integer | Durée du délai de grâce quand activé |
| Mode | Selection | `immediate` / `delayed` / `grace_period` |

## Calendrier — week-ends

| Paramètre | Type | Rôle |
|---|---|---|
| `block_on_saturday` | Boolean | Interdit les suspensions le samedi |
| `block_on_sunday` | Boolean | Interdit les suspensions le dimanche |
| `activation_allowed_weekend` | Boolean | Autorise les activations/réactivations le week-end |

## Calendrier — jours fériés

| Paramètre | Type | Rôle |
|---|---|---|
| `block_on_holidays` | Boolean | Interdit les suspensions les jours fériés |
| `activation_on_holidays` | Boolean | Autorise les activations/réactivations les jours fériés |
| `holiday_country_mode` | Selection | Pays utilisé pour la détection des jours fériés. La détection s'appuie sur `customer.country_id` de la fiche client. Pays cibles v1 : **France métropolitaine, DOM-TOM (Guadeloupe, Martinique, Saint-Martin, Guyane — calendriers locaux), Mali, Sénégal**. L'ajout d'un nouveau pays se fait par configuration (source de calendrier), sans modification de code métier. |

## Report manuel

| Champ (par client) | Type | Rôle |
|---|---|---|
| `cutoff_postponed_until` | Date | Repousse la suspension du client à la date indiquée, indépendamment des règles globales (CAP-10) |

## Comportement attendu

- Toute modification d'un paramètre prend effet **au prochain cycle du scheduler** sans redéploiement ni redémarrage.
- Les paramètres de calendrier (jours interdits) ne s'appliquent **qu'aux actions de suspension**. Les activations/réactivations ne sont calendaires que si `activation_allowed_weekend` ou `activation_on_holidays` est explicitement désactivé — par défaut elles sont autorisées 24/7 (cf. CAP-11).
- Un report manuel (`cutoff_postponed_until`) prime sur toute évaluation automatique tant que la date n'est pas dépassée.

## Logique d'éligibilité de suspension

Le contrat fonctionnel attendu est :

1. Si `suspension_enabled = False` → pas de suspension automatique, point.
2. Si le jour courant est samedi et `block_on_saturday = True` → suspension reportée.
3. Si le jour courant est dimanche et `block_on_sunday = True` → suspension reportée.
4. Si le jour courant est férié dans le pays du client et `block_on_holidays = True` → suspension reportée.
5. Si un `cutoff_postponed_until` existe et est dans le futur → suspension reportée à cette date.
6. Sinon, suspension exécutée à l'heure configurée après écoulement du délai/grâce.
