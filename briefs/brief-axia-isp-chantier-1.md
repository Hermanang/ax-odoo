---
title: Brief Chantier 1 — Synchronisation Odoo ↔ Splynx
project: AXIA ISP Management Suite
chantier: 1
audience: Équipe développement externe
status: Ready for handoff
date: 2026-06-11
version: 3.0
prerequisite: brief-axia-isp-commun.md
chantier-prerequisite: Chantier 0 signé-off
---

# Brief Chantier 1 — Synchronisation Odoo ↔ Splynx

> 📖 **Document compagnon** : ce brief se lit **avec** [`brief-axia-isp-commun.md`](./brief-axia-isp-commun.md). **Lis-le d'abord, c'est important.**
>
> 🔒 **Prérequis** : Le Chantier 0 (Sprint de Fondation) est accepté et signé-off. Les modules audit + RBAC, le secret store, les golden files Splynx et la CI sont en place et opérationnels.

## Sommaire

1. [Objet du chantier](#1-objet-du-chantier)
2. [Scope](#2-scope)
3. [Acceptance globale](#3-acceptance-globale)
4. [Les 23 stories](#4-les-23-stories)
5. [Tests d'acceptation Chantier 1](#5-tests-dacceptation-chantier-1)

---

## 1. Objet du chantier

Ce chantier, c'est le cœur du réacteur : on livre le connecteur **bidirectionnel** Odoo ↔ Splynx complet. Au menu : création conditionnelle des clients, propagation des modifications, webhooks sécurisés, réconciliation, lecture monitoring temps réel, et mapping des statuts contractuels.

---

## 2. Scope

Voilà ce qu'on embarque dans ce chantier :

- La configuration par marque (1 backend Splynx par société Odoo)
- Un client HTTP minimaliste avec retry, timeout et métriques
- La couche Mapper versionnée (v1, future v2…) construite à partir des golden files du Chantier 0
- Le pattern Outbox pour l'idempotence des appels sortants
- Les bindings + jobs asynchrones (création + mise à jour customer & internet-service, hors PPP)
- La détection d'éligibilité côté Odoo + les indicateurs visuels (`splynx_status`)
- La cascade matching anti-doublons (niveaux 1 + 2 actifs, niveau 3 fuzzy en feature flag, désactivé par défaut)
- La sync incrémentale stricte (no-op si rien n'a changé)
- Le field-level ownership (détection + écrasement tracé)
- Le webhook entrant avec vérification HMAC + anti-replay + inbox + dispatcher asynchrone
- Le cron de réconciliation horaire (notre filet de sécurité pour les webhooks perdus)
- Le mapping et la machine d'états des statuts contractuels (4 statuts natifs Splynx, jamais de suppression physique)
- Le rapport de divergence quotidien
- La configuration de la fairness queue multi-tenant
- Les tests E2E des scénarios d'acceptance PRD §15.1 et §15.6

---

## 3. Acceptance globale

Le chantier est bon pour le sign-off **uniquement si** ces 8 conditions sont remplies :

1. ✅ Un client créé dans Odoo (avec une commande ISP confirmée) apparaît dans Splynx en **< 5 min p95**, **< 15 min p99**.
2. ✅ Une modification d'un champ Odoo-owned propage à Splynx selon ownership en moins de 5 min p95.
3. ✅ Un webhook Splynx reçu est vérifié HMAC, dédoublonné par `event_id`, traité de manière asynchrone et la réponse HTTP est < 500 ms.
4. ✅ Un cron de réconciliation horaire détecte les divergences entre Odoo et Splynx et émet des événements `reconciliation.discrepancy.detected`.
5. ✅ La lecture de l'état réseau (IPv4/v6, NAS, uptime) est à jour dans Odoo avec indicateur de fraîcheur.
6. ✅ Le matching anti-doublons niveaux 1 + 2 est opérationnel ; le niveau 3 fuzzy est désactivé par défaut (feature flag).
7. ✅ Les statuts contractuels Odoo sont mappés sur les 4 statuts natifs Splynx (`new` / `active` / `blocked` / `inactive`) avec transitions invalides refusées et **zéro `DELETE` physique** émis (audit grep).
8. ✅ Le test E2E reproduisant le scénario PRD §15.1 (création conditionnelle, 5 sous-scénarios) passe sur les 4 backends.

---

## 4. Les 23 stories

> Chaque story suit le format `As a / I want / So that`, avec des ACs comportementaux en Given/When/Then, la couverture FR/NFR/ADR et l'effort. Allez, on détaille :

### Story E1.S1 — Modèle de configuration par marque

**As a** admin fonctionnel,
**I want** un modèle de configuration de backend (1 record par marque) qui référence l'URL de l'instance Splynx, la version du Mapper utilisée, les références aux secrets (token API et secret HMAC webhook) dans le secret store, et un indicateur de santé,
**So that** chaque marque a son backend isolé configurable sans dev.

**Référence architecture** : ADR-007 (authentification API Splynx).
**Effort** : 3 jours.

**Acceptance Criteria** :
- **Given** un admin Odoo, **When** il crée un backend rattaché à une société, **Then** le record est créé avec les références aux secrets dans le secret store,
- **And** un user d'une autre société ne voit pas ce backend (isolation par record rule),
- **And** un test de connectivité « Ping » est exposé en bouton UI, accessible uniquement aux administrateurs Odoo.

---

### Story E1.S2 — Client HTTP unifié

**As a** dev backend connecteur,
**I want** un client HTTP sortant uniforme avec connection pooling, timeout configurable, retry exponentiel sur 5xx + 429 (3 essais), chargement lazy du token depuis le secret store, et logging structuré JSON,
**So that** tous les appels Splynx sont observables et résilients (les SLAs de latence sont atteignables).

**Référence architecture** : ADR-008 (client HTTP `httpx`).
**Effort** : 4 jours.

**Acceptance Criteria** :
- **Given** un backend configuré, **When** le code instancie le client, **Then** le token est chargé depuis le secret store à la première requête,
- **And** chaque appel HTTP est en TLS 1.2+ minimum, avec en-tête `Authorization` ne fuitant jamais dans les logs,
- **And** un appel rencontrant un HTTP 503 est retenté 3 fois avec backoff exponentiel,
- **And** chaque appel logge en JSON `{logger, correlation_id, backend, endpoint, duration_ms, status}` sans secret ni mot de passe PPP,
- **And** des métriques de latence et de taux d'erreur par backend sont exposées (consommables par Prometheus en Chantier 4),
- **And** les tests unitaires mockent **toutes** les routes (zéro appel réseau en CI).

---

### Story E1.S3 — Mapper v1 pour customer

**As a** dev backend connecteur,
**I want** une interface Mapper abstraite + une implémentation v1 qui transforme un partner Odoo en payload Splynx pour `POST customer` conformément aux golden files du Chantier 0,
**So that** le Mapper peut évoluer en v2 sans refactor du code appelant (versioning par bump majeur).

**Référence architecture** : ADR-001 (contrat Mapper & POC payloads Splynx).
**Effort** : 3 jours.

**Acceptance Criteria** :
- **Given** un partner Odoo avec les champs identitaires standard (nom, adresse, téléphone, email, coordonnées géo, catégorie),
- **When** le Mapper v1 le transforme,
- **Then** le payload retourné correspond aux golden files capturés en Chantier 0 (assertion structurelle après normalisation des identifiants),
- **And** un test de régression échoue si un champ du golden disparaît ou change de nom,
- **And** l'ID Odoo du partner est traçable dans le payload Splynx (champ `additional_attributes` ou équivalent selon ADR-001 — contrat Mapper & POC payloads Splynx).

---

### Story E1.S4 — Outbox transactionnelle (idempotence)

**As a** dev backend connecteur,
**I want** une table outbox qui mémorise pour chaque opération sortante un hash du payload et la réponse cachée,
**So that** un retry après crash worker ne crée pas de doublon côté Splynx (pas d'`Idempotency-Key` natif Splynx — R10).

**Effort** : 3 jours.

**Acceptance Criteria** :
- **Given** un job qui pousse un customer vers Splynx, **When** le job termine avec succès, **Then** une ligne outbox `(backend, endpoint, payload_hash, response_cached, timestamp)` est insérée **dans la même transaction**,
- **Given** le job est rejoué avec le même payload (crash + reprise), **When** il s'exécute, **Then** l'outbox détecte le hash existant, **et n'émet pas** l'appel Splynx, retourne la réponse cachée,
- **And** un événement d'audit `customer.synced.replayed` est émis.

---

### Story E1.S5 — Binding customer + job de synchronisation

**As a** dev backend connecteur,
**I want** un modèle binding (`partner` ↔ `external_id Splynx`) + un job asynchrone de synchronisation,
**So that** la création d'un partner éligible déclenche une création côté Splynx au prochain tick du connecteur, sans flag manuel.

**Effort** : 4 jours.

**Acceptance Criteria** :
- **Given** un partner avec au moins une commande ISP confirmée ou une souscription active (cf. règles d'éligibilité dans la SPEC), **When** le tick connecteur détecte l'éligibilité, **Then** une ligne binding est créée (sans `external_id` initial),
- **And** un job de synchronisation est enqueué sur un channel isolé par société (cf. Story S22),
- **When** le job s'exécute, **Then** il pousse le customer vers Splynx via le Mapper v1,
- **And** la réponse `external_id` Splynx est stockée dans le binding,
- **And** un événement d'audit `customer.created` est émis (avec correlation_id),
- **And** la latence p95 entre éligibilité détectée et création Splynx effective est ≤ 5 min (mesurée).

---

### Story E1.S6 — Statut Splynx calculé + bandeau d'avertissement

**As a** opérateur,
**I want** un champ `splynx_status` calculé sur le partner (valeurs : `not_eligible`, `eligible_pending`, `synced`, `error`), filtrable et groupable en vue liste, et un bandeau d'avertissement visible quand le partner est `eligible_pending`,
**So that** je vois immédiatement quels clients vont impacter la licence Splynx au prochain run.

**Effort** : 3 jours.

**Acceptance Criteria** :
- **Given** un partner sans contrat ISP ni binding, **Then** statut `not_eligible` (couleur neutre),
- **Given** un partner devenu éligible mais sans binding encore créé, **Then** statut `eligible_pending` (couleur attention) + bandeau visible expliquant l'impact licence,
- **Given** un binding avec `external_id` rempli, **Then** statut `synced` + lien vers la fiche Splynx,
- **Given** une dernière tentative de sync échouée, **Then** statut `error` + dernier message d'erreur consultable.

---

### Story E1.S7 — Bouton « Forcer création Splynx » gated

**As a** admin fonctionnel,
**I want** un bouton « Forcer création » sur la fiche partner, visible uniquement aux administrateurs Odoo, avec saisie obligatoire d'un motif (≥ 10 caractères),
**So that** je peux pousser dans Splynx des cas exceptionnels (migration legacy, client provisionné sans vente Odoo) avec trace.

**Effort** : 2 jours.

**Acceptance Criteria** :
- **Given** un user Commercial, **Then** le bouton est invisible,
- **Given** un user administrateur, **When** il clique sur le bouton, **Then** un wizard demande un motif obligatoire,
- **When** il valide, **Then** un binding est créé immédiatement, un job de sync est enqueué, un événement d'audit `splynx.force_creation` est émis (acteur, motif, partner).

---

### Story E1.S8 — Propagation write-time des modifications client

**As a** opérateur,
**I want** qu'une modification d'un champ Odoo-owned sur un partner déjà synchronisé déclenche un job de propagation vers Splynx,
**So that** Splynx reflète la modification en ≤ 5 min p95.

**Effort** : 4 jours.

**Acceptance Criteria** :
- **Given** un partner synchronisé, **When** l'opérateur modifie un champ Odoo-owned (ex. email), **Then** le système recompute la signature du payload Mapper,
- **And** si la signature diffère de la dernière connue, un job de propagation est enqueué,
- **And** si la signature est identique (aucun changement effectif), **aucun job** n'est enqueué,
- **And** un événement d'audit `customer.updated` est émis,
- **And** un test de charge mesure le SLA p95 ≤ 5 min sur 100 modifications consécutives.

---

### Story E1.S9 — Cascade matching anti-doublons (niveaux 1 + 2)

**As a** dev backend connecteur,
**I want** que toute création Splynx exécute en cascade le matching niveau 1 (identifiant Splynx connu côté Odoo) puis niveau 2 (email normalisé ET mobile au format E.164) avant le POST,
**So that** aucun doublon n'est créé en cas de migration ou de re-binding.

**Effort** : 4 jours.

**Acceptance Criteria** :
- **Given** un partner éligible dont l'identifiant Splynx est déjà connu (cas migration), **When** la cascade s'exécute, **Then** le système bascule en mise à jour (pas de POST), le binding est créé avec cet identifiant,
- **Given** un partner éligible sans identifiant Splynx connu mais dont l'email + mobile normalisés correspondent à un client Splynx existant, **When** la cascade s'exécute, **Then** mise à jour, événement d'audit `customer.dedup.matched` émis avec contexte (niveau et champs comparés),
- **Given** un partner éligible sans aucun match, **Then** POST de création exécuté.

---

### Story E1.S10 — Niveau 3 fuzzy + queue d'arbitrage humaine

**As a** opérateur senior,
**I want** un niveau 3 de matching fuzzy (similarité de nom > 0.85 ET même code postal ET même pays) **désactivé par défaut**, activable par paramètre, qui bloque la création Splynx et liste le cas dans un écran « doublons potentiels » pour arbitrage manuel,
**So that** je gère les cas ambigus sans casser la sync automatique.

**Effort** : 5 jours.

**Acceptance Criteria** :
- **Given** le feature flag désactivé (défaut), **Then** le niveau 3 ne s'exécute jamais,
- **Given** flag activé et un partner avec similarité de nom > 0.85 à un client Splynx existant, **When** la cascade s'exécute, **Then** la création est bloquée, un record d'arbitrage est créé (partner Odoo, candidat Splynx, score, champs comparés),
- **And** un écran liste les cas en attente, **et** un opérateur peut trancher : merge / créer quand même / ignorer,
- **And** chaque arbitrage émet un événement d'audit `customer.dedup.resolved`.

---

### Story E1.S11 — Sync incrémentale stricte (no-op)

**As a** ops,
**I want** qu'un cycle de sync programmé sur un client non modifié n'émette **aucun appel d'écriture** vers Splynx, vérifiable dans les métriques,
**So that** le no-op est garanti à 100 000 abonnés (pas de DOS du tenant Splynx).

**Effort** : 2 jours.

**Acceptance Criteria** :
- **Given** 1 000 partners synchronisés sans modification, **When** le tick connecteur s'exécute, **Then** **aucun** appel de mise à jour n'est émis vers Splynx (vérifié par compteur),
- **And** aucun événement d'audit `customer.updated` n'est inséré,
- **And** un test de charge mesure que 10 000 partners non modifiés sont scannés en < 30 s.

---

### Story E1.S12 — Field-level ownership : détection + écrasement tracé

**As a** dev backend connecteur,
**I want** un service qui à chaque cycle compare la valeur Odoo et Splynx pour les champs à propriété conjointe (cf. matrice PRD §6) et applique la valeur du propriétaire en cas de divergence avec audit,
**So that** la matrice d'ownership est respectée et chaque écrasement est tracé.

**Couvre** : **Note** : ownership PPP traité en Chantier 2.
**Effort** : 3 jours.

**Acceptance Criteria** :
- **Given** un champ Odoo-owned (ex. email) divergent entre Odoo et Splynx, **When** le cycle de sync s'exécute, **Then** la valeur Odoo est poussée vers Splynx,
- **And** un événement d'audit `field.overwrite` est émis avec `{field, source, before, after}`,
- **Given** un champ Splynx-owned (ex. IPv4 courante) divergent, **When** le cycle s'exécute, **Then** la valeur Splynx écrase la valeur Odoo et un événement `field.overwrite` est émis (source = splynx).

---

### Story E1.S13 — UI lecture seule sur champs Splynx-owned

**As a** opérateur,
**I want** que les champs Splynx-owned (état réseau temps réel : IPv4/v6, NAS, uptime) soient visuellement marqués lecture seule dans l'UI Odoo,
**So that** je ne tente pas de modifier des valeurs qui seront écrasées au prochain cycle.

**Effort** : 1 jour.

**Acceptance Criteria** :
- **Given** la vue formulaire d'un service Internet, **Then** les champs Splynx-owned sont affichés en lecture seule avec un indicateur visuel,
- **And** la modification via UI est impossible,
- **And** une tentative via JSON-RPC est techniquement possible mais écrasée au cycle suivant (test E2E).

---

### Story E1.S14 — Binding service Internet + job de synchronisation

**As a** dev backend connecteur,
**I want** un binding pour le service Internet (analogue au binding customer) avec sync à Splynx,
**So that** un service Internet créé dans Odoo (après validation contrat) apparaît dans Splynx avec ses attributs techniques (sauf PPP, qui arrive en Chantier 2).

**Effort** : 3 jours.

**Acceptance Criteria** :
- **Given** un service Internet Odoo lié à un customer synchronisé, **When** le tick connecteur détecte le nouveau service, **Then** un binding est créé,
- **And** un job de synchronisation pousse le service vers Splynx (champs PPP laissés vides — provisionnés au Chantier 2),
- **And** un événement d'audit `service.created` est émis.

---

### Story E1.S15 — Webhook entrant : controller + HMAC + inbox + anti-replay

**As a** dev backend connecteur,
**I want** un controller HTTP entrant qui vérifie la signature HMAC-SHA1 en constant-time sur le raw body, dépose la requête dans une table inbox, et répond 200 OK en < 500 ms,
**So that** Splynx peut notifier Odoo des changements (état réseau, paiement, statut) avec sécurité forte.

**Référence architecture** : ADR-006 (sécurité webhooks Splynx).
**Effort** : 4 jours.

**Acceptance Criteria** :
- **Given** un POST sur le controller webhook avec body JSON et en-tête de signature, **When** le controller traite la requête,
- **Then** la signature HMAC-SHA1 est calculée sur le **raw body** (jamais re-sérialisé) avec le secret du backend correspondant chargé depuis le secret store,
- **And** la comparaison est en constant-time (anti-timing),
- **Given** signature invalide, **Then** réponse 401, événement d'audit `webhook.rejected.signature` émis,
- **Given** signature valide et `event_id` jamais vu, **Then** ligne inbox insérée (unicité par `(event_id, backend)`), réponse 200 OK en < 500 ms,
- **Given** signature valide mais `event_id` déjà vu (replay), **Then** insert rejeté par l'unicité, réponse 200 idempotente, événement `webhook.duplicate`,
- **Given** webhook avec timestamp > 5 min dans le passé, **Then** rejet, événement `webhook.rejected.replay`.

---

### Story E1.S16 — Traitement asynchrone des webhooks (dispatcher)

**As a** dev backend connecteur,
**I want** un job asynchrone qui consomme l'inbox webhook et dispatche par type d'événement (customer modifié → sync inverse, statut service modifié → mise à jour binding, paiement reçu → trigger réactivation préparé pour Chantier 3),
**So that** le controller webhook répond immédiatement et le traitement réel est asynchrone, tracé, observable.

**Effort** : 4 jours.

**Acceptance Criteria** :
- **Given** une ligne inbox non traitée, **When** le job tourne, **Then** le handler enregistré pour ce type d'événement est appelé,
- **And** un événement d'audit `webhook.received` est émis avec le type,
- **And** un marker `processed_at` est positionné sur la ligne inbox,
- **Given** un type d'événement inconnu (futur Splynx), **Then** audit `webhook.received` avec marker `unhandled` + alerte ops (le job **ne lève pas** d'exception).

---

### Story E1.S17 — Cron de réconciliation horaire (filet de sécurité webhooks perdus)

**As a** ops,
**I want** un cron horaire qui pour chaque backend récupère la liste paginée des customers Splynx modifiés depuis le dernier run et compare avec l'état Odoo,
**So that** les webhooks perdus silencieusement sont détectés et rattrapés en < 1 h.

**Effort** : 5 jours.

**Acceptance Criteria** :
- **Given** le cron horaire, **When** il s'exécute, **Then** pour chaque backend, la liste des customers Splynx modifiés depuis le dernier run est récupérée (avec pagination ; fallback `id BETWEEN` si Splynx n'expose pas de filtre `updated_since` natif — OQ-3),
- **And** chaque ligne est comparée à la signature stockée côté binding,
- **And** chaque divergence émet un événement `reconciliation.discrepancy.detected` (binding, champs divergents),
- **And** un job de re-sync est enqueué pour rattraper la divergence selon ownership,
- **And** une métrique de comptage des divergences sur 24 h est exposée (consommable Chantier 4).

---

### Story E1.S18 — Lecture monitoring + indicateur de fraîcheur

**As a** agent support N1/N2,
**I want** que la fiche service Internet affiche l'état réseau temps réel (statut RADIUS, IPv4/v6, NAS, uptime) avec horodatage et indicateur de fraîcheur (vert ≤ 5 min, attention 5-30 min, alerte > 30 min),
**So that** je résous 95 % de mes tickets sans ouvrir Splynx.

**Effort** : 2 jours.

**Acceptance Criteria** :
- **Given** un service avec dernière mise à jour il y a 2 min, **Then** indicateur vert, affichage « il y a 2 min »,
- **Given** dernière mise à jour il y a 12 min, **Then** indicateur attention,
- **And** un bouton « Rafraîchir manuellement » déclenche une lecture ad-hoc Splynx, accessible aux rôles habilités (cf. RBAC Chantier 0).

---

### Story E1.S19 — Mapping statuts contractuels + machine d'états

**As a** dev backend connecteur,
**I want** un mapping strict entre les statuts Odoo et les 4 statuts natifs Splynx (`new`, `active`, `blocked`, `inactive`) avec une machine d'états qui refuse les transitions invalides,
**So that** la machine d états des statuts contractuels est conforme et le scénario d'acceptance §15 (cf. SPEC) est satisfait.

**Couvre** : **Réf** : `service-states.md`.
**Effort** : 3 jours.

**Acceptance Criteria** :
- **Given** un service en statut `inactive`, **When** Odoo tente une transition `inactive` → `active`, **Then** la transition est refusée avec un message clair (un service résilié doit passer par un nouveau cycle `new`),
- **Given** un service en `new`, **When** transition vers `active`, **Then** OK, propagation Splynx,
- **And** un événement d'audit `service.statechange` est émis,
- **And** **aucun appel `DELETE`** n'est jamais émis (vérifié par audit du code).

---

### Story E1.S20 — Remontée des changements de statut Splynx + alerte écart

**As a** ops,
**I want** qu'un changement de statut technique Splynx (ex. action manuelle d'un agent réseau) déclenche un webhook traité par Odoo qui met à jour le statut côté Odoo selon ownership et lève une alerte si l'écart est sémantiquement significatif,
**So that** la remontée des statuts Splynx vers Odoo est correcte et l'asymétrie blocage manuel est détectée.

**Effort** : 3 jours.

**Acceptance Criteria** :
- **Given** un service Splynx passe en `blocked` suite à une action manuelle (pas Odoo), **When** le webhook est reçu et traité, **Then** le statut Odoo est mis à jour,
- **And** comme aucun job de suspension Odoo n'a été émis dans les 60 min précédentes, une alerte `manual_block_detected` est levée (visible dans un écran ops),
- **And** un événement d'audit `service.statechange.manual` est émis.

---

### Story E1.S21 — Rapport divergence quotidien + alerte email

**As a** admin fonctionnel,
**I want** un rapport quotidien automatique listant les partners avec contrat ISP actif sans binding Splynx (faux négatifs) ET les bindings Splynx sans contrat ISP actif Odoo (faux positifs), avec alerte email si au moins un cas est trouvé,
**So that** je détecte les désynchros silencieuses.

**Effort** : 2 jours.

**Acceptance Criteria** :
- **Given** 3 partners éligibles sans binding, **When** le cron quotidien tourne, **Then** un rapport est généré listant les 3 cas,
- **And** un email est envoyé à l'opérateur configuré.

---

### Story E1.S22 — Fairness multi-tenant queue asynchrone

**As a** ops,
**I want** que la configuration de la queue asynchrone garantisse l'isolation entre marques (une marque congestionnée ne doit jamais bloquer les autres) et la priorité des actions critiques (paiement, suspension) sur les syncs courantes,
**So that** la fairness multi-tenant et la priorité asymétrique sont respectées.

**Référence architecture** : ADR-009 (fairness queue multi-tenant).
**Effort** : 1 jour.

**Acceptance Criteria** :
- **Given** la configuration choisie par votre équipe, documentée dans `architecture.md` (cf. note §5.1 brief commun), **When** Odoo démarre, **Then** chaque société dispose d'un channel isolé pour ses syncs,
- **And** une marque congestionnée (50 jobs en backlog) **ne ralentit pas** les jobs des autres marques (mesure : latence p95 des jobs des autres marques inchangée),
- **And** les actions critiques (paiement / suspension préparé pour Chantier 3) ont une priorité supérieure aux syncs courantes,
- **And** la convention de nommage des channels est documentée pour les ops (runbook).

---

### Story E1.S23 — Tests E2E acceptance §15.1 + §15.6

**As a** QA,
**I want** un test end-to-end qui reproduit les scénarios PRD §15.1 (création conditionnelle, 5 sous-scénarios) et §15.6 (conflit de sync bidirectionnel) sur une instance de test connectée à un tenant Splynx simulé (golden files Chantier 0),
**So that** le SLA Chantier 1 est démontré avant le handoff au Chantier 2.

**Couvre** : §15.1, §15.6.
**Effort** : 5 jours.

**Acceptance Criteria** :
- **Given** une instance de test avec 4 backends configurés (HTTP mockés à partir des golden files), **When** la suite E2E tourne, **Then** les 5 sous-scénarios §15.1 passent (prospect, vente confirmée, force creation, tentative non-admin, rapport divergence),
- **And** les scénarios §15.6 (Odoo-owned divergent, Splynx-owned divergent, aucune divergence) passent,
- **And** la mesure SLA p95 entre éligibilité et création Splynx est < 5 min sur 100 cycles.

---

## 5. Tests d'acceptation Chantier 1

🟡 **À nous envoyer avant de démarrer** : les tests red-phase ATDD qui couvrent les 8 critères d'acceptance globale (§3) et chaque AC de story. Voilà la couverture cible :

| Catégorie | Stories sources |
|---|---|
| Modèle backend + isolation | S1 |
| Client HTTP (retry, TLS, logging) | S2 |
| Mapper v1 (régression golden files) | S3 |
| Outbox idempotence (replay sans doublon) | S4 |
| Binding & job customer + SLA p95 | S5, S11 |
| `splynx_status` + bandeau | S6, S7 |
| Propagation write-time + SLA | S8 |
| Cascade dedup (3 niveaux) | S9, S10 |
| Field-level ownership | S12, S13 |
| Binding service + machine d'états | S14, S19 |
| Webhook : HMAC, anti-replay, dispatcher | S15, S16 |
| Réconciliation cron | S17 |
| Lecture monitoring + fraîcheur | S18 |
| Remontée changements Splynx + alerte | S20 |
| Rapport divergence + email | S21 |
| Fairness multi-tenant (isolation + priorité) | S22 |
| E2E §15.1 + §15.6 | S23 |

---

**Et voilà pour le Brief Chantier 1 !**
