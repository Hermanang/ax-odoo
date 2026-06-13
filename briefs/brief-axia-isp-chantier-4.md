---
title: Brief Chantier 4 — Production-ready (observabilité + ops)
project: AXIA ISP Management Suite
chantier: 4
audience: Équipe développement externe
status: Ready for handoff
date: 2026-06-11
version: 3.0
prerequisite: brief-axia-isp-commun.md
chantier-prerequisite: Chantier 3 signé-off
---

# Brief Chantier 4 — Production-ready (observabilité + ops)

> 📖 **Document compagnon** : utilise ce brief **avec** [`brief-axia-isp-commun.md`](./brief-axia-isp-commun.md). **Lis-le d'abord.**
>
> 🔒 **Prérequis** : Chantier 3 (Workflow impayés & billing) accepté et signé-off. Le workflow d'impayés bout-en-bout, le moteur de suspension, les calendaires multi-pays et la réactivation 24/7 tournent déjà.

## Sommaire

1. [Objet du chantier](#1-objet-du-chantier)
2. [Scope](#2-scope)
3. [Acceptance globale](#3-acceptance-globale)
4. [Les 15 stories](#4-les-15-stories)
5. [Tests d'acceptation Chantier 4](#5-tests-dacceptation-chantier-4)

---

## 1. Objet du chantier

On y est, le dernier chantier. Celui qui transforme un bon logiciel en un service vraiment **exploitable en production sur la durée**. On met en place toute la couche **observabilité + ops** : exposition Prometheus complète, alerting Alertmanager par défaut, dashboard santé Odoo natif, logs structurés JSON consommables par Loki, archivage Parquet S3 trimestriel immuable (WORM), purge automatique au-delà de 5 ans, wizards RGPD (export et anonymisation), runbooks ops complémentaires, health check externe, hook Sentry. Le Saint Graal de la mise en prod, pour faire simple.

---

## 2. Scope

Voici ce qu'on livre concrètement :

- Un endpoint Prometheus `/axia/metrics` qui expose les métriques opérationnelles (queue, latence Splynx, taux d'erreur, fraîcheur sync, conflits, PPP legacy, etc.)
- Des métriques détaillées par canal de queue (backlog, échecs, latence) et par appel API Splynx (latence histogramme + taux d'erreur 24 h)
- Des règles Alertmanager par défaut (backlog critique, error rate, sync stale, jobs bloqués, réactivation en retard)
- La garantie de logs structurés JSON partout dans le code (scan CI bloquant les usages directs de `logging.getLogger` ou `print()`)
- Un dashboard kanban natif dans Odoo « Santé connecteur » (1 carte par marque, refresh 30 s)
- Un test E2E + scan static garantissant la propagation end-to-end du `correlation_id`
- Un job trimestriel d'archivage **Parquet S3** des partitions audit > 12 mois (idempotent)
- L'application de l'**Object Lock WORM 5 ans** sur tous les uploads Parquet (immutabilité native côté stockage)
- La purge automatique des partitions > 5 ans avec méta-audit (rétention 5 ans glissants stricte)
- Un wizard d'**export RGPD** par période / société (CSV)
- Un wizard d'**anonymisation RGPD** (droit à l'effacement) — jamais de suppression physique
- Les runbooks ops 02 (jobs bloqués), 03 (rotation token Splynx), 05 (failover multi-DB)
- Un health check controller `/axia/healthz` pour les probes externes
- Un hook Sentry sur les jobs en échec (non-retryable)

---

## 3. Acceptance globale

On considère le chantier livré quand ces 9 points sont validés :

1. ✅ L'endpoint `/axia/metrics` expose bien toutes les métriques opérationnelles attendues (latence, queue, error rate, fraîcheur, conflits, PPP legacy) au format Prometheus standard.
2. ✅ Les règles Alertmanager par défaut tournent et sont testées (backlog critique, error rate, sync stale, jobs bloqués > 30 min, réactivation en retard).
3. ✅ Tous les logs sont au format JSON structuré avec `correlation_id`, consommables par un agrégateur (Loki). Le scan CI bloque bien l'usage direct de `logging.getLogger` ou `print()` dans tout le code AXIA.
4. ✅ Le dashboard kanban Odoo « Santé connecteur » est opérationnel (4 cartes, refresh 30 s).
5. ✅ Le job trimestriel d'archivage Parquet vers S3 avec Object Lock WORM 5 ans fonctionne (test : DELETE et overwrite rejetés en 403 après upload).
6. ✅ La purge automatique > 5 ans est documentée et tracée (méta-audit `axia.audit.purged`).
7. ✅ Les wizards RGPD sont opérationnels : export par période / société (CSV, 100k events en < 5 min) + anonymisation par abonné (pas de DELETE physique côté Odoo ni Splynx).
8. ✅ Les runbooks 02, 03, 05 sont écrits et dry-runnés par un dev non auteur (signature `validated_by` dans chaque runbook).
9. ✅ Le health check probe externe est opérationnel et le hook Sentry actif sur les jobs en échec.

---

## 4. Les 15 stories

### Story E4.S1 — Endpoint Prometheus

**As a** ops,
**I want** un controller HTTP `GET /axia/metrics` qui expose au format Prometheus standard l'ensemble des métriques opérationnelles AXIA (latence queue, latence Splynx, taux d'erreur, fraîcheur de la sync, compteurs PPP legacy, etc.),
**So that** un serveur Prometheus externe peut scraper toutes les 15 secondes.

**Référence architecture** : ADR-010 (observabilité J1).

**Acceptance Criteria** :
- **Given** un Prometheus configuré pour scraper l'endpoint, **When** un scrape arrive, **Then** la réponse est au format `text/plain; version=0.0.4` avec l'ensemble des métriques attendues,
- **And** une IP allow-list au reverse proxy en amont restreint l'accès (cohérent avec la sécurité webhook),
- **And** la durée du scrape reste < 5 s même à 100 000 abonnés.

---

### Story E4.S2 — Métriques de queue asynchrone

**As a** ops,
**I want** des métriques par canal de queue : nombre de jobs en attente, échecs sur la dernière heure, durée maximale des jobs « started » (pour détecter les blocages),
**So that** je détecte un backlog critique et les jobs bloqués.


**Acceptance Criteria** :
- Métriques exposées avec un label par canal,
- Test : enqueuer 50 jobs sur le canal critique de suspension, la métrique de pending remonte à 50 puis décroît à 0 au fil du traitement.

---

### Story E4.S3 — Métriques API Splynx (latence histogramme + taux d'erreur)

**As a** ops,
**I want** un histogramme de latence par backend / endpoint / méthode / statut HTTP + une gauge du taux d'erreur 24 h par backend,
**So that** je détecte une dégradation de l'API d'un tenant Splynx.


**Acceptance Criteria** :
- L'histogramme utilise des buckets utiles aux SLAs (par exemple `[0.1, 0.5, 1, 2, 5, 10]` secondes),
- Test : 100 requêtes simulées (95 OK, 5 erreurs), les métriques sont cohérentes avec les comptages attendus.

---

### Story E4.S4 — Règles Alertmanager par défaut

**As a** ops,
**I want** des fichiers d'exemple `prometheus.yml.example` + `alertmanager.yml.example` contenant les règles d'alerting opérationnelles (backlog critique > 50, error rate > 5 %, sync stale > 1 h, jobs `started_for` > 30 min, réactivation en retard > 10 min),
**So that** un déploiement reproduit l'alerting opérationnel sans devoir le réinventer.


**Acceptance Criteria** :
- Les fichiers d'exemple sont valides (vérifiés par `promtool check rules`),
- Chaque règle est accompagnée d'une note d'opération : raison de la règle, seuil choisi, action ops attendue.

---

### Story E4.S5 — Logs structurés JSON imposés partout

**As a** dev backend,
**I want** que le wrapper logger (livré au Chantier 0) soit utilisé partout dans le projet, avec une sortie JSON structurée consommable par un agrégateur (Loki),
**So that** la cohérence des logs end-to-end est garantie et le scan CI bloque toute régression.


**Acceptance Criteria** :
- **Given** une PR contenant un `logging.getLogger` direct ou un `print()` dans un module AXIA, **Then** la CI échoue (grep bloquant),
- **Given** un cycle complet de décision exécuté, **Then** les logs produits sont au format JSON et contiennent `correlation_id`, `customer_id` (ou contextuel), `level`, `module`.

---

### Story E4.S6 — Dashboard kanban Odoo « Santé connecteur »

**As a** opérateur,
**I want** un dashboard kanban natif dans Odoo, avec 1 carte par backend (Marque A, Marque B, Marque C, Marque D) affichant : backlog actuel, taux d'échec sur la dernière heure, latence p95 Splynx, dernière sync OK, pourcentage de PPP legacy actifs, alertes courantes,
**So that** un opérateur voit la santé du système sans devoir ouvrir Grafana externe.


**Acceptance Criteria** :
- 4 cartes (une par marque),
- Refresh testé toutes les 30 s,
- Code couleur cohérent avec les seuils d'alerting (ex. attention si latence > 1 s, alerte si > 5 s).

---

### Story E4.S7 — `correlation_id` propagé end-to-end (test + scan static)

**As a** ops,
**I want** un test end-to-end qui exécute un cycle complet (impayé → suspension → paiement → réactivation) en vérifiant que le `correlation_id` est constant sur l'ensemble des événements émis, **et** un scan static qui détecte tout enqueue de job ne propageant pas le `correlation_id`,
**So that** le debug end-to-end depuis un seul ID reste possible.


**Acceptance Criteria** :
- Le test passe avec assertion : le `correlation_id` est constant sur ≥ 12 événements émis dans le cycle,
- Le scan static détecte tout `delay()` (ou équivalent) sans propagation du `correlation_id` dans les metadata du job (warning bloquant la PR).

---

### Story E4.S8 — Archivage trimestriel Parquet vers S3

**As a** ops,
**I want** un script lancé par cron trimestriel qui exporte les partitions audit > 12 mois en Parquet vers S3,
**So that** la rétention chaud / froid (12 mois en PG / 5 ans en S3) est opérationnelle.

**Référence architecture** : ADR-005 (schéma audit & partitionnement).

**Acceptance Criteria** :
- **Given** une partition d'audit datant de plus de 12 mois, **When** le job tourne, **Then** la partition est exportée en Parquet vers une clé S3 datée (ex. `s3://<bucket>/<année>/<mois>/events.parquet`),
- **And** le schéma Parquet est aligné avec les colonnes de la table partitionnée,
- **And** un marker d'idempotence est positionné (re-run du job n'exporte pas deux fois la même partition).

---

### Story E4.S9 — Object Lock WORM sur les uploads Parquet

**As a** ops,
**I want** que chaque upload Parquet vers S3 soit configuré avec Object Lock mode COMPLIANCE et une rétention de 5 ans,
**So that** l'immutabilité 5 ans est garantie nativement au niveau du stockage.


**Acceptance Criteria** :
- **Given** un upload Parquet effectué, **When** une tentative de DELETE est faite sur l'objet, **Then** la réponse est `403 AccessDenied`,
- **And** une tentative d'overwrite du même objet est refusée,
- **And** un script de vérification post-upload valide l'immutabilité.

---

### Story E4.S10 — Purge automatique > 5 ans + méta-audit

**As a** ops,
**I want** un cron quotidien qui détache et supprime les partitions audit dont l'âge dépasse 5 ans (et marque les objets S3 équivalents comme expirés), avec un méta-événement d'audit pour tracer l'opération,
**So that** la rétention 5 ans glissants stricte est opérationnelle.


**Acceptance Criteria** :
- **Given** une partition datant de 5 ans + 1 jour, **When** le cron tourne, **Then** la partition est détachée puis droppée,
- **And** un événement d'audit `axia.audit.purged` est émis (période purgée, nombre d'événements concernés),
- **And** un test « fast-forward » de 5 ans simulé déclenche le drop attendu.

---

### Story E4.S11 — Wizard export RGPD période / société

**As a** opérateur RGPD,
**I want** un wizard qui pour une période et une société donne en CSV l'ensemble des événements d'audit AXIA (sync, suspensions, paiements, réactivations…),
**So that** les obligations légales d'export comptable et RGPD (droit d'accès) sont satisfaites.


**Acceptance Criteria** :
- Le wizard expose les champs : date de début, date de fin, société, types d'événements (multi-sélection),
- Le CSV généré contient les colonnes d'audit + le payload aplati par type d'événement,
- L'export de 100 000 événements se termine en < 5 min,
- L'export complet (toutes sociétés) est gated « Direction Groupe » ; l'export d'une société spécifique est gated selon le périmètre du rôle.

---

### Story E4.S12 — Wizard anonymisation droit à l'effacement

**As a** opérateur RGPD,
**I want** un wizard qui pour un abonné anonymise sa fiche Odoo (remplace les champs personnels par des valeurs neutres) et force le statut `inactive` côté Splynx, **sans aucune suppression physique**,
**So that** le droit à l'effacement est appliqué tout en préservant l'intégrité de l'audit.


**Acceptance Criteria** :
- Les champs personnels (nom, email, mobile, adresse) sont remplacés par des valeurs neutres déterministes (par exemple `anon_<hash8>`),
- Le client Splynx est passé à `inactive`, **pas supprimé**,
- Un événement d'audit `customer.anonymized` est émis (avec un hash de la valeur originale pour preuve sans révélation),
- Aucun DELETE physique n'est émis ni côté Odoo, ni côté Splynx (vérifié par grep code + assertion sur les appels API).

---

### Story E4.S13 — Runbooks ops complémentaires (jobs bloqués, rotation token, failover multi-DB)

**As a** ops on-call,
**I want** trois runbooks markdown : récupération des jobs bloqués au-delà de 30 min, rotation du token API Splynx, bascule depuis multi-company vers multi-DB,
**So that** toutes les procédures critiques sont scriptées et exécutables en 15 min par un on-call non familier.


**Acceptance Criteria** :
- Chaque runbook suit le format défini en Chantier 2 (contexte, symptômes, diagnostic, procédure, rollback, post-mortem checklist),
- Un dry-run par un dev non auteur valide chaque procédure (signature `validated_by`).

---

### Story E4.S14 — Health check controller + probes externes

**As a** ops,
**I want** un endpoint `GET /axia/healthz` qui retourne 200 + statut JSON si Odoo, la base de données, le cache Redis et au moins un backend Splynx sont joignables ; 503 sinon,
**So that** un probe externe (Pingdom, healthchecks.io, etc.) peut surveiller la disponibilité.


**Acceptance Criteria** :
- État nominal → 200 OK,
- Redis inaccessible → 503,
- Tous les backends Splynx inaccessibles → 503 (mais pas si un seul est down — le système est partiellement opérationnel),
- Rate-limit appliqué (par exemple 60 requêtes / min / IP) pour éviter l'abus.

---

### Story E4.S15 — Hook Sentry sur les jobs en échec

**As a** ops,
**I want** un hook sur la queue asynchrone qui pour chaque job en échec non-retryable envoie l'exception, le `correlation_id` et un contexte minimum à Sentry (DSN chargé depuis le secret store),
**So that** le mode pionnier d'instrumentation J1 est complet et chaque erreur production est tracée.


**Acceptance Criteria** :
- **Given** une `FailedJobError` levée, **When** le hook s'exécute, **Then** Sentry reçoit l'erreur avec les tags `correlation_id`, `backend`, type d'événement,
- **And** une assertion vérifie qu'aucun secret n'est transmis dans le payload Sentry.

---

## 5. Tests d'acceptation Chantier 4

🟡 **Avant de démarrer, on attend** : les tests red-phase ATDD qui couvrent les 9 critères d'acceptance globale (§3) et chaque AC story. Couverture cible :

| Catégorie | Stories sources |
|---|---|
| Endpoint Prometheus + perf scrape | S1 |
| Métriques queue + Splynx API | S2, S3 |
| Règles Alertmanager validées | S4 |
| Logs JSON imposés + grep CI | S5 |
| Dashboard kanban Odoo | S6 |
| Propagation `correlation_id` E2E | S7 |
| Archivage Parquet S3 trimestriel idempotent | S8 |
| WORM Object Lock (DELETE / overwrite rejetés) | S9 |
| Purge automatique > 5 ans + méta-audit | S10 |
| Wizard export RGPD période / société (perf 100k) | S11 |
| Wizard anonymisation sans DELETE physique | S12 |
| Runbooks 02 / 03 / 05 validés par dry-run | S13 |
| Health check probe externe + comportement 503 | S14 |
| Hook Sentry sans fuite secrets | S15 |

---

**Et voilà, c'est la fin du Brief Chantier 4.**

> Une fois ce chantier accepté, le système AXIA ISP Management Suite est **production-ready** pour de bon : observabilité complète, archivage immuable, RGPD opérationnel, runbooks dry-runnés, supervision externe en place. Bon courage pour ce dernier chantier, et rendez-vous en prod. 🏗️
