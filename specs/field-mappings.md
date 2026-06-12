# Mappings de champs Odoo ↔ Splynx

Matrice des correspondances de champs entre Odoo et Splynx, par sens de synchronisation. Référencé par les capacités CAP-1, CAP-4, CAP-5, CAP-6 du SPEC.

> 🔒 **Identifiant PPP & Mot de passe PPP** : ces champs sont **Odoo-owned** — Odoo génère, Splynx applique. Voir `sync-ownership.md` pour le détail de la matrice d'ownership.

## Données client (sens Odoo → Splynx)

| Odoo | Splynx |
|---|---|
| first_name | first_name |
| last_name | last_name |
| street | street_1 |
| zip | zip_code |
| city | city |
| phone | phone |
| mobile | mobile |
| email | email |
| geo_lat | latitude |
| geo_lng | longitude |

**Règle** : sync uniquement si changement détecté côté Odoo (cf. CAP-3).

## Recherche de doublon avant création

Critères matchés (cf. CAP-2) :

- Name
- Last name
- email
- téléphone
- customer_number
- external_id

Comportement :

| Cas | Action |
|---|---|
| Client existant dans Splynx | Update |
| Client inexistant | Create |

## Statuts contractuels (sens Odoo → Splynx)

| Champ custom Odoo | Splynx |
|---|---|
| En attente de validation du contrat | `draft` |
| Actif | `Activated` |
| Suspendu pour non-paiement | `blocked` |
| Résilier | `terminated` |

**Règle critique** : la résiliation **NE supprime PAS** le client côté Splynx. Splynx passe à `terminated` mais l'enregistrement persiste (CAP-5 + contrainte associée).

## Données service Internet (sens Odoo → Splynx)

| Odoo | Splynx |
|---|---|
| Identifiant PPP | User ppp |
| Mot de passe PPP | Pass ppp |
| Service | Service |
| IP Static | Enable / disabled |
| Serial Number router | S/N |

## Données de monitoring (sens Splynx → Odoo, lecture seule)

| Donnée Splynx | Champ custom Odoo |
|---|---|
| online/offline | radius_status |
| current_ipv4 | current_ipv4 |
| current_ipv6 | current_ipv6 |
| NAS | nas_name |
| uptime | uptime |

Ces champs alimentent la vue support/diagnostic dans Odoo (CAP-6). Aucune écriture retour vers Splynx.

## Conditions de création dans Splynx

La création du client dans Splynx (cf. CAP-1) est soumise à deux conditions cumulatives sur la fiche Odoo :

- `create_to_radius` = `True`
- `radius_connection_type` ≠ `False` (un type de connexion doit être sélectionné)

Sinon, aucune création n'est émise vers Splynx, même si le client existe côté Odoo.
