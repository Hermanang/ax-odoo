# Politique de synchronisation et résolution de conflit

Matrice d'ownership : pour chaque champ synchronisé entre Odoo et Splynx, définit **quel système fait foi** en cas de divergence détectée à un cycle de sync. Référencée par les capacités CAP-1, CAP-3, CAP-4, CAP-5, CAP-6 du SPEC.

## Principe

**Field-level ownership** : chaque champ a un propriétaire désigné. En cas de divergence détectée à un cycle de sync, la valeur du propriétaire écrase celle de l'autre côté, et l'événement d'écrasement est tracé (CAP-15).

Le principe colle à la dichotomie de sources de vérité gravée en contrainte du SPEC : Odoo = commercial / métier, Splynx = technique réseau.

## Matrice d'ownership

### Champs propriété d'Odoo (Odoo gagne)

| Champ Odoo | Champ Splynx | Direction effective | Raison |
|---|---|---|---|
| first_name | first_name | Odoo → Splynx | Identité commerciale |
| last_name | last_name | Odoo → Splynx | Identité commerciale |
| street | street_1 | Odoo → Splynx | Coordonnées commerciales |
| zip | zip_code | Odoo → Splynx | Coordonnées commerciales |
| city | city | Odoo → Splynx | Coordonnées commerciales |
| phone | phone | Odoo → Splynx | Coordonnées commerciales |
| mobile | mobile | Odoo → Splynx | Coordonnées commerciales |
| email | email | Odoo → Splynx | Coordonnées commerciales |
| geo_lat | latitude | Odoo → Splynx | Donnée géographique métier |
| geo_lng | longitude | Odoo → Splynx | Donnée géographique métier |
| Statut contractuel (En attente / Actif / Suspendu / Résilier) | draft / Activated / blocked / terminated | Odoo → Splynx | Décision métier (cf. [[service-states]]) |
| cutoff_postponed_until | (sans équivalent Splynx) | Odoo uniquement | Décision opérateur, n'est pas synchronisée |
| IP Static (flag) | Enable / disabled | Odoo → Splynx | Décision de provisioning |
| Service (type) | Service | Odoo → Splynx | Décision commerciale |
| Serial Number router | S/N | Odoo → Splynx | Inventaire métier |
| Identifiant PPP | User ppp (PPP login) | Odoo → Splynx | **Généré par Odoo.** Unicité garantie par backend Splynx (vérification anti-collision avant propagation) |
| Mot de passe PPP | Pass ppp (PPP password) | Odoo → Splynx | **Généré par Odoo.** Stocké chiffré at-rest côté Odoo (le clair n'existe qu'en mémoire au moment de la propagation) |

### Champs propriété de Splynx (Splynx gagne, lecture seule côté Odoo)

| Champ Splynx | Champ Odoo | Direction effective | Raison |
|---|---|---|---|
| online/offline | radius_status | Splynx → Odoo | État technique de session |
| current_ipv4 | current_ipv4 | Splynx → Odoo | Attribué dynamiquement par Splynx |
| current_ipv6 | current_ipv6 | Splynx → Odoo | Attribué dynamiquement par Splynx |
| NAS | nas_name | Splynx → Odoo | Donnée d'infrastructure réseau |
| uptime | uptime | Splynx → Odoo | Métrique réseau temps réel |

## Règles opérationnelles

1. **Détection de divergence** : à chaque cycle de sync, comparer la dernière valeur connue côté Odoo et côté Splynx pour les champs à propriété conjointe (les champs Odoo→Splynx).
2. **Si divergence sur un champ Odoo-owned** : la valeur Odoo écrase Splynx ; un événement "écrasement Splynx par Odoo" est journalisé (champ, ancienne valeur Splynx, nouvelle valeur Odoo, horodatage).
3. **Si divergence sur un champ Splynx-owned** : la valeur Splynx écrase Odoo ; même journalisation symétrique. Ces champs ne sont jamais éditables manuellement côté Odoo (UI lecture seule).
4. **Sync uniquement si changement** (CAP-3) : un champ inchangé des deux côtés ne déclenche aucun appel d'écriture. Comparaison sur hash ou valeur brute selon disponibilité.
5. **Champs sans équivalent** : `cutoff_postponed_until` (Odoo only) et tous les paramètres administrables ne sont jamais poussés à Splynx.
6. **Aucune réécriture inutile** : si la valeur du propriétaire est égale à celle de l'autre côté, aucun appel n'est émis.

## Cas spécial : édition manuelle d'un champ Splynx-owned côté Odoo

L'UI doit rendre ces champs en lecture seule. Si malgré tout un script ou un import externe modifie un champ Splynx-owned côté Odoo, le prochain cycle de sync écrase la valeur Odoo par celle de Splynx (Splynx gagne) et journalise l'écrasement. Aucun blocage applicatif n'est appliqué — la convention est portée par l'UI et la trace d'audit.

## Cas spécial : édition manuelle d'un champ Odoo-owned côté Splynx

Symétriquement, si un admin Splynx modifie une coordonnée commerciale (ex. téléphone), le prochain cycle de sync écrasera la valeur Splynx par celle d'Odoo. La modification Splynx est donc transitoire. L'opérateur doit modifier ces champs côté Odoo.

**Cas particulier PPP** — Pour `User ppp` / `Pass ppp`, toute édition manuelle côté Splynx provoque **deux ruptures PPPoE successives** (l'édition + l'écrasement Odoo au cycle suivant). Règle stricte : **aucune édition PPP côté Splynx, jamais** — utiliser la procédure de rotation Odoo (cf. brief Chantier 2).

## Évolutivité

Si un nouveau champ à synchroniser est introduit (par exemple un service IPTV en v2), il doit être ajouté à cette matrice **avant** son implémentation, avec décision explicite de propriétaire.
