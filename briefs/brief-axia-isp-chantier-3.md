---
title: Brief Chantier 3 — Workflow impayés & billing automation
project: AXIA ISP Management Suite
chantier: 3
audience: Équipe développement externe
status: Ready for handoff
date: 2026-06-11
version: 3.0
prerequisite: brief-axia-isp-commun.md
chantier-prerequisite: Chantier 2 signé-off
---

# Brief Chantier 3 — Workflow impayés & billing automation

> 📖 **Document compagnon** : ce brief s'utilise **avec** [`brief-axia-isp-commun.md`](./brief-axia-isp-commun.md). **Lis-le d'abord.**
>
> 🔒 **Prérequis** : Chantier 2 (PPP & actions techniques) accepté et signé-off. Les actions techniques `suspend` / `reactivate` / `terminate` doivent être opérationnelles côté connecteur.

## Sommaire

1. [Objet du chantier](#1-objet-du-chantier)
2. [Scope](#2-scope)
3. [Acceptance globale](#3-acceptance-globale)
4. [Les 20 stories](#4-les-20-stories)
5. [Tests d'acceptation Chantier 3](#5-tests-dacceptation-chantier-3)

---

## 1. Objet du chantier

On s'attaque au gros morceau : le **workflow d'impayés bout-en-bout**. Le principe est simple — un cron qui tourne toutes les 15 minutes pour repérer les factures en retard, une évaluation selon des paramètres métier configurables sans code (mode immédiat / différé / grâce, délais, calendriers pays), l'application des règles calendaires multi-pays (FR métropole + DOM-TOM + Mali + Sénégal), la suspension automatique via le connecteur, la réactivation 24/7 dès qu'un paiement tombe, le report manuel par client, et la possibilité de tout couper avec une suspension globale. Et cerise sur le gâteau : l'écran d'administration sans code pour que les opérateurs métier soient autonomes.

---

## 2. Scope

- **Module paramètres admin** — rien de moins qu'un modèle d'extension de la configuration Odoo native, avec paramètres par société, validation à la saisie, tooltips, aperçu de l'effet et audit des changements
- **Service calendrier multi-pays** — jours fériés FR métropole + DOM-TOM (971, 972, 973, MF, BL) + Mali + Sénégal, le tout via `python-holidays`, avec gestion du samedi/dimanche
- **Détection cron court** — un cron toutes les 15 minutes qui scanne les factures en retard, garanti idempotent (pas de doublon, on te voit venir)
- **Moteur de suspension** — 3 modes au choix (`immediate`, `delayed`, `grace_period`), délais paramétrables
- Application des **règles calendaires** avec report au prochain jour autorisé (et audit du motif, bien sûr)
- **Report manuel** par client (`cutoff_postponed_until`) — ce champ prime sur les règles auto
- **Exécution de suspension** asynchrone via un canal critique haute priorité (SLA ≤ 5 min p95)
- **Réactivation 24/7** déclenchée sur paiement, via un canal critique distinct
- **Désactivation explicite** des réactivations le weekend et les jours fériés (pour l'opérateur prudent qui préfère attendre le lundi)
- **Bascule arrière** — si la facture est régularisée entre-temps, on annule la suspension en attente (logique, non ?)
- **Suspension globale** désactivable d'un clic via paramètre (gestion de crise ou maintenance, pas besoin de toucher au code)
- **Audit E2E** complet avec `correlation_id` propagé sur toute la chaîne décisionnelle, re-jouabilité réglementaire
- Tests E2E qui reproduisent fidèlement les scénarios d'acceptance : §15.2 (Mali), §15.3 (Guadeloupe 27 mai), §15.4 (réactivation 24/7), §15.5 (report manuel), §15.7 (multi-société)

---

## 3. Acceptance globale

Accroche-toi, le chantier est accepté **uniquement si** :

1. ✅ Scénario PRD §15.3 (Guadeloupe 27 mai) : suspension calculée le 27 mai 2026 (férié `971`) est reportée au 28 mai, événement `suspension.postponed.calendar` émis (motif `holiday`, pays `971`).
2. ✅ Scénario PRD §15.2 (Mali, `grace_period` 10 jours) : cycle complet **détection → suspension J+10 → paiement → réactivation** s'exécute **sans intervention manuelle**.
3. ✅ Scénario PRD §15.7 (multi-société) : société FR et société SN coexistent **sans contamination**, paramètres différents par société respectés.
4. ✅ Modification d'un paramètre métier (mode, délai, calendaire) prend effet **au prochain cycle scheduler** (≤ 15 min), aucun redémarrage worker requis.
5. ✅ Scénario PRD §15.5 (report manuel) : `cutoff_postponed_until` prime sur les règles auto tant que la date est dans le futur, l'évaluation normale reprend automatiquement à expiration.
6. ✅ Suspension globale désactivable via paramètre booléen (les réactivations restent actives).
7. ✅ Sur 30 cycles consécutifs mesurés : latence détection → décision ≤ 15 min, latence suspension après éligibilité ≤ 5 min p95, latence réactivation après paiement ≤ 5 min p95.

---

## 4. Les 20 stories

C'est dense mais tout est là — chaque brique du workflow a sa story.

### Story E3.S1 — Module paramètres admin sans code

**As a** dev backend,
**I want** un modèle d'extension de la configuration Odoo qui expose **par société** tous les paramètres métier listés dans `admin-parameters.md` (mode, délais, calendaires, options de réactivation, etc.),
**So that** un admin fonctionnel peut éditer toutes ces valeurs depuis l'UI sans intervention dev, et la modification prend effet au prochain cycle sans redémarrage.

**Effort** : 3 jours.

**Acceptance Criteria** :
- **Given** un admin dans la société XIWO, **When** il modifie `Mode = grace_period` et sauve, **Then** la valeur est persistée pour XIWO uniquement,
- **And** un user d'une autre société voit la valeur par défaut inchangée pour sa propre société (isolation par société),
- **And** aucun redémarrage worker n'est nécessaire (le process worker reste up),
- **And** la liste exhaustive des paramètres exposés correspond à celle de `admin-parameters.md` (cf. SPEC).

---

### Story E3.S2 — Audit des changements de paramètres

**As a** ops,
**I want** que chaque modification d'un paramètre admin génère un événement d'audit (paramètre, ancienne valeur, nouvelle valeur, auteur, horodatage),
**So that** la traçabilité 5 ans est garantie sur la configuration métier.

**Effort** : 2 jours.

**Acceptance Criteria** :
- **Given** `Mode = delayed` et un admin le change en `grace_period`, **When** il sauve, **Then** un événement d'audit `admin.parameter.changed` est émis avec le delta complet `{paramètre, ancienne, nouvelle}`,
- **And** un `correlation_id` distinct est attribué par soumission (groupe de changements dans la même action).

---

### Story E3.S3 — Validation saisie + tooltips

**As a** admin fonctionnel,
**I want** chaque paramètre accompagné d'une étiquette en français + tooltip clair expliquant son effet métier + validation à la saisie (entier ≥ 0, sélection bornée, date future pour les champs de report),
**So that** une saisie invalide affiche un message d'erreur clair sans sauvegarder.

**Effort** : 2 jours.

**Acceptance Criteria** :
- **Given** un admin saisit `suspension_delay_value = -3`, **Then** erreur affichée « Le délai doit être ≥ 0 », pas de sauvegarde,
- **Given** il survole le paramètre « block_on_holidays », **Then** un tooltip affiche un texte explicatif clair, ex. « Si activé, aucune suspension le jour férié — la date sera reportée au prochain jour autorisé. »

---

### Story E3.S4 — Aperçu de l'effet immédiat

**As a** admin fonctionnel,
**I want** que l'écran paramètres affiche un aperçu de l'effet attendu après modification (ex. « avec ces paramètres, une facture détectée en retard aujourd'hui sera suspendue le {date calculée} »),
**So that** je valide mes choix avant de sauver.

**Effort** : 3 jours.

**Acceptance Criteria** :
- **Given** mode `delayed`, délai 15 jours, calendrier FR métropole, **When** l'admin modifie le délai à 8 jours, **Then** le label d'aperçu affiche immédiatement la nouvelle date calculée (ex. « Facture détectée le 10 juin 2026 → suspension prévue le 18 juin, ajustée au prochain jour autorisé selon le calendrier »).

---

### Story E3.S5 — Valeurs par défaut alignées sur la pratique marché FR

**As a** admin fonctionnel,
**I want** qu'à l'installation, les paramètres aient les valeurs par défaut suivantes (cf. `admin-parameters.md`) : mode `delayed`, délai 15 jours calendaires, période de grâce désactivée, blocage les jours fériés activé, blocage les samedis activé, réactivation 24/7 activée, fuzzy dedup désactivé,
**So that** la configuration par défaut est immédiatement conforme aux usages métier français.

**Effort** : 1 jour.

**Acceptance Criteria** :
- **Given** installation fraîche d'une société, **Then** les 13 valeurs par défaut listées dans `admin-parameters.md` sont positionnées correctement (test exhaustif sur chaque clé).

---

### Story E3.S6 — Service calendrier multi-pays (jours fériés + DOM-TOM)

**As a** dev backend billing,
**I want** un service qui pour un triplet `(date, pays, subdivision)` retourne un booléen « jour férié » + le nom du jour férié, couvrant **FR métropole, Guadeloupe (971), Martinique (972), Guyane (973), Saint-Martin (MF), Saint-Barthélemy (BL), Mali (ML), Sénégal (SN)**,
**So that** le moteur de suspension peut décider du report selon le calendrier local du client.

**Effort** : 3 jours.

**Acceptance Criteria** :
- **Given** date 27 mai 2026 et subdivision Guadeloupe (971), **When** la fonction est appelée, **Then** retourne `True` avec nom « Abolition de l'esclavage »,
- **Given** date 14 juillet 2026 et pays Sénégal, **Then** retourne `False` (pas un jour férié sénégalais),
- **And** une couverture de test existe pour chacun des 8 pays / subdivisions du catalogue v1.

---

### Story E3.S7 — Détection des impayés via cron court

**As a** dev backend billing,
**I want** un cron qui toutes les 15 minutes scanne les factures atteignant le statut configuré comme déclencheur (`overdue_invoice_status`, défaut `late`) et émet un événement par facture éligible nouvellement détectée,
**So that** la latence détection → décision est < 15 min et la détection est idempotente.

**Effort** : 3 jours.

**Acceptance Criteria** :
- **Given** une facture bascule en `late` à 14h00, **When** le cron tourne à 14h15, **Then** un événement `invoice.overdue.detected` est émis pour cette facture, et une entrée de décision est créée (statut « en attente d'évaluation »),
- **And** le re-run du cron à 14h30 ne crée **pas** de doublon de détection pour la même facture (idempotence).

---

### Story E3.S8 — Moteur de suspension à 3 modes

**As a** dev backend billing,
**I want** un service qui évalue chaque décision en attente selon l'un des 3 modes paramétrés (`immediate`, `delayed`, `grace_period`) et calcule la date prévue de suspension,
**So that** un changement de mode prend effet au prochain cycle sans redéploiement.

**Effort** : 3 jours.

**Acceptance Criteria** :
- **Given** mode `delayed`, délai 15 jours, facture détectée le 1er juin, **Then** date prévue = 16 juin (jours calendaires, pas ouvrés),
- **Given** mode `immediate`, facture détectée à 14h, **Then** date prévue = 14h (immédiat),
- **Given** mode `grace_period`, délai 5 jours + grâce 10 jours, **Then** date prévue = J+15,
- **And** un changement du mode dans le paramétrage est pris en compte au prochain cycle d'évaluation (≤ 15 min) sans redémarrage.

---

### Story E3.S9 — Application des règles calendaires + report

**As a** dev backend billing,
**I want** qu'avant l'exécution d'une suspension, un service vérifie si la date calculée tombe samedi / dimanche / férié selon les paramètres et le pays du client, et la reporte au **prochain jour autorisé** avec audit du motif,
**So that** les obligations réglementaires multi-pays sont respectées et le scénario Guadeloupe 27 mai passe.

**Effort** : 4 jours.

**Acceptance Criteria** :
- **Given** une suspension calculée le 27 mai 2026 pour un client Guadeloupe, **When** le service applique les règles calendaires, **Then** la suspension est reportée au 28 mai,
- **And** un événement d'audit `suspension.postponed.calendar` est émis avec `{motif: 'holiday', country: '971', date_initial, date_reprogrammed, nom_du_férié}`,
- **Given** une suspension calculée samedi et le paramètre « bloquer le samedi » activé, **Then** la suspension est reportée au lundi,
- **And** la boucle de report est bornée (max 14 jours en avant) pour éviter une boucle infinie en cas de mauvaise configuration.

---

### Story E3.S10 — Report manuel par client

**As a** opérateur,
**I want** un champ `cutoff_postponed_until` sur la fiche client qui désactive l'évaluation des règles automatiques tant que la date est dans le futur, et reprend automatiquement à expiration,
**So that** je peux geler manuellement un cas particulier sans toucher aux paramètres globaux.

**Effort** : 3 jours.

**Acceptance Criteria** :
- **Given** un client avec `cutoff_postponed_until = 2026-06-15`, **When** le cron tourne entre aujourd'hui et le 14 juin, **Then** un événement `invoice.overdue.detected` est tout de même émis (visibilité), la décision est marquée « reportée manuellement », **aucune suspension** n'est planifiée,
- **Given** le 16 juin, le cron tourne, **Then** un événement `cutoff.postponed.expired` est émis, l'évaluation normale reprend, une suspension est planifiée si la facture est toujours impayée.

---

### Story E3.S11 — Exécution de la suspension via canal critique

**As a** dev backend billing,
**I want** un job qui à la date prévue exécute l'action `suspend` côté connecteur (livré en Chantier 2), met à jour le statut Odoo, et journalise,
**So that** la latence suspension après éligibilité validée est ≤ 5 min p95.

**Effort** : 3 jours.

**Acceptance Criteria** :
- **Given** une décision arrivée à sa date prévue, **When** le job tourne sur un canal de queue à priorité haute (cf. fairness §5.1 brief commun), **Then** l'action `suspend` côté connecteur est appelée,
- **And** le statut côté binding Odoo passe à `blocked`,
- **And** un événement d'audit `suspension.executed` est émis, avec le `correlation_id` partagé depuis l'événement initial `invoice.overdue.detected`,
- **And** la latence mesurée entre la date prévue et l'exécution effective est ≤ 5 min p95 sur 100 cycles.

---

### Story E3.S12 — Réactivation 24/7 sur paiement

**As a** dev backend billing,
**I want** un déclencheur sur paiement (`account.payment` à l'état posté) qui pour les clients suspendus avec leur facture régularisée, enqueue immédiatement un job de réactivation sur un canal de queue à priorité haute, **24 heures sur 24 et 7 jours sur 7**,
**So that** la latence réactivation après paiement est ≤ 5 min p95 et la cible best-in-class FR est atteinte.

**Effort** : 3 jours.

**Acceptance Criteria** :
- **Given** un client suspendu et un paiement reçu un samedi à 22h, **When** le paiement est posté, **Then** un job de réactivation est enqueué immédiatement,
- **And** l'action `unblock` côté connecteur est appelée,
- **And** le statut côté binding Odoo passe à `active`,
- **And** un événement d'audit `reactivation.executed` est émis,
- **And** la latence mesurée entre le paiement posté et la réactivation effective est ≤ 5 min p95 sur 100 cycles.

---

### Story E3.S13 — Désactivation explicite des réactivations weekend / férié

**As a** admin,
**I want** que si les paramètres `activation_allowed_weekend = False` ou `activation_on_holidays = False` sont positionnés, les réactivations soient reportées au prochain jour ouvré local avec audit,
**So that** les opérateurs avec un choix conservateur (pas d'opération hors heures ouvrées) sont supportés.

**Effort** : 2 jours.

**Acceptance Criteria** :
- **Given** `activation_allowed_weekend = False` et un paiement reçu samedi, **When** la réactivation est évaluée, **Then** elle est reportée au lundi à 08h,
- **And** un événement d'audit `activation.postponed.calendar` est émis avec le motif.

---

### Story E3.S14 — Bascule arrière facture régularisée

**As a** dev backend billing,
**I want** que si une facture éligible à suspension est régularisée (payée, annulée, modifiée) avant l'exécution de la suspension, le job en attente soit annulé et un événement émis,
**So that** aucun client qui a déjà payé n'est suspendu (cohérence métier critique).

**Effort** : 2 jours.

**Acceptance Criteria** :
- **Given** une suspension prévue à J+10 et un paiement reçu à J+5, **When** le paiement est posté, **Then** la décision en attente passe à l'état « annulée »,
- **And** le job en attente côté queue est marqué annulé (pas d'exécution),
- **And** un événement d'audit `suspension.cancelled` est émis avec le motif `payment_received`.

---

### Story E3.S15 — Suspension globale du workflow

**As a** admin,
**I want** un paramètre `suspension_enabled = False` qui désactive globalement toutes les suspensions automatiques (les réactivations restent actives),
**So that** je peux gérer une crise opérationnelle (panne réseau, maintenance majeure) sans intervention dev.

**Effort** : 2 jours.

**Acceptance Criteria** :
- **Given** `suspension_enabled = False` pour la société XIWO, **When** une suspension est due, **Then** elle n'est pas exécutée,
- **And** un événement d'audit `suspension.disabled_globally` est émis,
- **And** les réactivations sur paiement continuent de fonctionner normalement,
- **Given** le paramètre rebascule à `True`, **Then** les décisions en attente reprennent au prochain cycle d'évaluation.

---

### Story E3.S16 — Audit complet E2E + re-jouabilité

**As a** ops,
**I want** que pour un client donné, l'historique d'audit puisse reconstituer la **chaîne complète** d'un cycle d'impayé (`invoice.overdue.detected` → `suspension.scheduled` → `suspension.postponed.calendar` ou report manuel → `suspension.executed` ou `cancelled` → `payment.received` → `reactivation.executed`) avec le contexte décisionnel pour chaque étape,
**So that** la re-jouabilité réglementaire est garantie et le debug d'une réclamation client se fait en quelques minutes.

**Effort** : 3 jours.

**Acceptance Criteria** :
- **Given** un cycle complet exécuté, **When** un opérateur ouvre la vue « Chronologie audit client », **Then** la chaîne d'événements est visible avec un `correlation_id` constant,
- **And** chaque événement contient son contexte (mode actif, délai, calendrier, paramètres en vigueur au moment précis de la décision).

---

### Story E3.S17 — Test E2E §15.3 — scénario Guadeloupe 27 mai

**As a** QA,
**I want** un test E2E qui reproduit le scénario PRD §15.3 (client Guadeloupe, facture impayée, calcul de suspension tombant sur le 27 mai férié, reportée au 28 mai) sur instance de test,
**So that** la dimension calendaires multi-pays est démontrée avant handoff.

**Effort** : 3 jours.

**Acceptance Criteria** :
- Le test passe end-to-end avec assertions strictes sur les événements émis (`suspension.postponed.calendar` avec motif `holiday` et pays `971`) et les dates calculées (initiale = 27 mai, reprogrammée = 28 mai).

---

### Story E3.S18 — Test E2E §15.2 — scénario Mali grace_period

**As a** QA,
**I want** un test E2E qui reproduit le scénario PRD §15.2 (client Mali, mode `grace_period` 10 jours, cycle complet J → J+10 suspension → paiement J+11 → réactivation),
**So that** le workflow E2E est démontré avec les latences mesurées sous cibles.

**Effort** : 3 jours.

**Acceptance Criteria** :
- Le test passe avec des latences mesurées inférieures aux cibles (≤ 5 min p95 pour la suspension et pour la réactivation après paiement).

---

### Story E3.S19 — Test E2E §15.7 — cohabitation multi-société

**As a** QA,
**I want** un test E2E qui démontre que la société FR (mode `grace_period` 15 jours, blocage le samedi activé) et la société SN (mode `immediate`) coexistent **sans contamination**,
**So that** l'isolation multi-société est démontrée bout-en-bout sur le workflow billing.

**Effort** : 3 jours.

**Acceptance Criteria** :
- Un user de la société FR ne voit aucun client SN (vérifié UI + JSON-RPC),
- La suspension du client SN s'exécute immédiatement, la suspension du client FR est reportée au lundi (samedi bloqué),
- L'audit est isolé par société.

---

### Story E3.S20 — Test E2E §15.4 + §15.5 — réactivation 24/7 + report manuel

**As a** QA,
**I want** deux tests E2E reproduisant les scénarios PRD §15.4 (réactivation un samedi à 22h) et §15.5 (le report manuel prime sur les règles auto, et l'évaluation reprend à expiration),
**So that** la couverture des flux billing critiques est complète.

**Effort** : 3 jours.

**Acceptance Criteria** :
- Les deux tests passent avec des assertions complètes sur les événements émis et les latences.

---

## 5. Tests d'acceptation Chantier 3

🟡 **À recevoir avant démarrage** : les tests red-phase ATDD couvrant les 7 critères d'acceptance globale (§3) et chaque AC story. En clair, avant de coder la première ligne, on veut le rouge. Couverture cible :

| Catégorie | Stories sources |
|---|---|
| Paramètres admin sans code (édition + isolation société) | S1, S5 |
| Audit des changements de paramètres | S2 |
| Validation saisie + tooltips + aperçu effet | S3, S4 |
| Service calendrier multi-pays (8 catalogues) | S6 |
| Détection cron court + idempotence | S7 |
| Moteur 3 modes (immediate / delayed / grace) | S8 |
| Application calendaires + report avec audit | S9 |
| Report manuel par client + expiration | S10 |
| Exécution suspension + SLA p95 | S11 |
| Réactivation 24/7 + SLA p95 | S12 |
| Désactivation réactivation weekend/férié | S13 |
| Bascule arrière facture régularisée | S14 |
| Suspension globale désactivable | S15 |
| Audit E2E + correlation_id complet | S16 |
| E2E §15.3 Guadeloupe | S17 |
| E2E §15.2 Mali grace_period | S18 |
| E2E §15.7 multi-société | S19 |
| E2E §15.4 + §15.5 réactivation + report manuel | S20 |

---

**Fin du Brief Chantier 3.**
