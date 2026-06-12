# États abonné / service — référentiel partagé

Définit les états reconnus du couple « abonné + service » et les transitions autorisées entre eux. Référencé par les capacités CAP-5, CAP-7 à CAP-13 du SPEC.

## États reconnus et mapping Splynx

| État Odoo (libellé fonctionnel) | Sens métier | Splynx | Source de vérité primaire |
|---|---|---|---|
| En attente de validation du contrat | Abonné/service créé mais pas encore activé (en cours de provisioning, vérifications, signature) | `draft` | Odoo |
| Actif | Service en cours, abonné autorisé à utiliser le réseau | `Activated` | Splynx (état technique) + Odoo (état contractuel) |
| Suspendu pour non-paiement | Service techniquement coupé (impayé, demande opérateur, autre règle) ; le contrat reste actif | `blocked` | Odoo décide, Splynx exécute |
| Résilier | Fin de contrat ; ni service ni facturation associée à venir | `terminated` | Odoo décide, Splynx exécute |

**Règle critique** : la résiliation **ne supprime pas** l'enregistrement client côté Splynx. L'enregistrement persiste avec statut `terminated`. Toute tentative de suppression physique est interdite (cf. constraint dédiée du SPEC).

L'état "suspendu" et l'état "résilié" sont déclenchés par une décision métier Odoo. Splynx peut également remonter un état technique correspondant (déconnecté, etc.) qui doit rester cohérent avec la décision métier ; toute divergence persistante est un incident.

## Transitions autorisées

- `en_attente` → `actif` : activation initiale par l'opérateur (provisioning OK, premier raccordement).
- `actif` → `suspendu` : décision automatique (workflow d'impayés, CAP-13) ou manuelle (opérateur).
- `suspendu` → `actif` : réactivation automatique (paiement reçu, fin de cause) ou manuelle. Autorisée 24/7 (CAP-11).
- `actif` → `résilié` : décision opérateur (fin de contrat, demande client).
- `suspendu` → `résilié` : décision opérateur après suspension durable.
- `en_attente` → `résilié` : annulation avant activation.

Transitions interdites :

- Pas de retour depuis `résilié` (un retour client donne lieu à un nouveau contrat / nouveau cycle `en_attente`).
- Pas de `en_attente` → `suspendu` direct (un service jamais activé ne peut être "suspendu").

## Règles de calendrier appliquées aux transitions

Seules les transitions `actif → suspendu` (entrée en suspension automatique) sont soumises au calendrier d'exécution défini en CAP-9 :

- Interdite les jours fériés du pays du client.
- Interdite les week-ends sauf si la règle pays l'autorise explicitement.
- Reportée à la prochaine fenêtre autorisée si la date calculée tombe sur un jour interdit.

Toutes les autres transitions (notamment `suspendu → actif`) sont autorisées 24/7.

## États techniques Splynx remontés (lecture seule côté Odoo)

Ces états ne sont pas dans la machine ci-dessus mais sont affichés en fiche client (CAP-6) :

- `online` / `offline` — session courante.
- `connecté avec IPv4 / IPv6 X.X.X.X` — IP affectée.
- `via NAS <id>` — équipement de terminaison.
- `uptime <durée>` — temps de session actif.

Ces informations sont consommées pour le support et le diagnostic ; elles ne déclenchent pas de transition d'état contractuel.
