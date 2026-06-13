---
stepsCompleted: [1, 2, 3, 4, 5, 6, 7, 8]
lastStep: 8
status: 'complete'
completedAt: '2026-06-10'
inputDocuments:
  - path: _bmad-output/planning-artifacts/prds/prd-axia-isp-2026-05-27/prd.md
    type: prd
    status: final-v2
  - path: _bmad-output/planning-artifacts/prds/prd-axia-isp-2026-05-27/addendum.md
    type: prd-addendum
  - path: _bmad-output/planning-artifacts/prds/prd-axia-isp-2026-05-27/.decision-log.md
    type: prd-decision-log
    entries: 27
  - path: _bmad-output/planning-artifacts/prds/prd-axia-isp-2026-05-27/review-edge-cases-ppp.md
    type: prd-review
  - path: _bmad-output/planning-artifacts/prds/prd-axia-isp-2026-05-27/review-rubric-v2.md
    type: prd-review
  - path: _bmad-output/specs/spec-axia-isp/SPEC.md
    type: spec-kernel
  - path: _bmad-output/specs/spec-axia-isp/sync-ownership.md
    type: spec-companion
    status: realigned-v2
  - path: _bmad-output/specs/spec-axia-isp/field-mappings.md
    type: spec-companion
    status: realigned-v2
  - path: _bmad-output/specs/spec-axia-isp/service-states.md
    type: spec-companion
  - path: _bmad-output/specs/spec-axia-isp/admin-parameters.md
    type: spec-companion
  - path: _bmad-output/specs/spec-axia-isp/glossary.md
    type: spec-companion
  - path: _bmad-output/planning-artifacts/cahier-des-charges/cdc-axia-isp-2026-05-27-v2.md
    type: cdc
    status: validated-client
  - path: inputs/specifications-client-2.md
    type: rights-matrix
  - path: _bmad-output/planning-artifacts/research/technical-splynx-api-connecteur-odoo-research-2026-05-27.md
    type: research
workflowType: 'architecture'
project_name: 'AXIA ISP Management Suite'
date: '2026-06-10'
language: 'fr'
---

# Architecture Solution — AXIA ISP Management Suite

Ce document, on l'a construit ensemble, étape par étape. Chaque section reflète une décision qu'on a prise collectivement, au fil des discussions architecturales. Prends-le comme une visite guidée du système plutôt qu'un cahier des charges rigide — tu y trouveras le pourquoi derrière chaque choix.

## Méta

- **Projet** : AXIA ISP Management Suite
- **Périmètre v1** : Internet / PPPoE / RADIUS, 4 backends Splynx (Marque A, Marque B, Marque C, Marque D)
- **Stack retenue** : Odoo 18 LTS + OCA connector + OCA queue_job + OCA keychain, PostgreSQL 15 + pg_partman, Docker on-premise, httpx custom, multi-company Odoo, audit `axia_audit_event`, WORM S3 Object Lock
- **Inversion ownership** : Odoo génère le PPP / Splynx applique (FR-11)
- **Langue** : français

---

## Project Context Analysis

### Requirements Overview

**Functional Requirements (83 FRs en 15 capabilities)** :
Synchronisation client/service Splynx ↔ Odoo (CAP-1 à CAP-6), workflow impayés automatisé bout-en-bout (CAP-7 à CAP-13), administration sans code (CAP-14), audit 5 ans glissants (CAP-15). Inversion CDC v2 : Odoo génère PPP, Splynx applique (FR-11/a/b/c, FR-78a). F7 RBAC à 7 rôles + délégation partenaires (FR-77 à FR-83). Multi-sociétés natif (FR-57 à FR-61) avec 4 backends Splynx étanches (Marque A, Marque B, Marque C, Marque D).

**Non-Functional Requirements (NFR-1 à NFR-25)** :

- **Réactivité** : NFR-2 propagation Odoo→Splynx ≤ 5 min p95 / ≤ 15 min p99 ; NFR-3 réactivation post-paiement ≤ 5 min p95 (24/7, cible best-in-class Free/Orange) ; NFR-1 détection impayé < 15 min ; NFR-4 suspension ≤ 5 min p95 ; NFR-5 monitoring ≤ 5 min p95 ; NFR-11 webhook < 500 ms.
- **Volumétrie & scalabilité** : NFR-6 100k abonnés actifs / ~1M opérations/mois ; NFR-7 sync incrémentale stricte (no-op si inchangé) ; NFR-8 pas de batch monolithique ; NFR-9 sharding logique `root.sync.<company_id>:1`.
- **Disponibilité** : NFR-10 99,9 % v1 (~8h45/an), cible 99,95 % v2 ; NFR-10a réactivation 24/7 ; NFR-13 reprise crash < 5 min.
- **Sécurité** : NFR-14 secrets OCA `keychain` + clé maître hors-base ; NFR-15 HMAC-SHA1 constant-time ; NFR-16 anti-replay UNIQUE INDEX event_id fenêtre 5 min ; NFR-17 TLS 1.2+ ; NFR-18 IP allow-list webhook ; NFR-25 `Pass ppp` + secrets chiffrés at-rest.
- **Observabilité J1 (mode pionnier)** : NFR-20 Prometheus (backlog, taux d'échec, latence p95, sync OK par backend) ; NFR-21 alerting ; NFR-22 logs JSON structurés ; NFR-23 dashboard Odoo santé connecteur ; NFR-24 correlation_id end-to-end.
- **Conformité** : NFR-19 RGPD audit multi-pays.
- **Worker** : NFR-12 queue_job dédié séparé de l'UI.

**Scale & Complexity** :

- Niveau : **enterprise** (intégration critique, multi-tenant, conformité réglementaire, audit 5 ans, observabilité pionnière)
- Domaine technique primaire : **backend / data sync / intégration**
- Composants architecturaux estimés : **6-8 modules Odoo** (core, connector Splynx, audit, RBAC, PPP, observability, billing-workflow, admin-params)
- Multi-marques : **4 backends Splynx étanches** (Marque A, Marque B, Marque C, Marque D), extensible nouvelles marques sans dev

### Technical Constraints & Dependencies

**API Splynx (R1, R3, R9, R10)** :

- Pas d'`Idempotency-Key` natif → Outbox applicatif Odoo `(payload_hash, response)` obligatoire.
- Pas d'`updated_since` documenté → cron checksum + fallback webhook (HMAC-SHA1, pas SHA256).
- Pas de champ `external_id` natif → pattern `additional_attributes.odoo_partner_id`.
- Statuts réels minuscules : `new` / `active` / `blocked` / `inactive`.
- Rate limits non documentés → channels queue_job throttle configurables + POC stress test.
- Webhooks sans garantie de livraison → cron de réconciliation horaire (mitigation R2).
- Asymétrie blocage manuel Splynx (R7) → runbook strict : aucune édition manuelle Splynx (rupture PPPoE massive sur PPP).

**Plateforme Odoo** :

- Odoo 18 LTS jusqu'à mi-2027 (R5 — versionning `BackendAdapter` + audit OCA trimestriel).
- OCA `connector` + `queue_job` + `keychain` (sizing dans addendum technique).
- Multi-company natif : 1 DB, N sociétés, 1 backend Splynx par société (R7-bis — bascule multi-DB triviale via `dbfilter` si OQ-20 l'impose).

**PostgreSQL 15 + pg_partman** :

- Partitionnement mensuel `axia_audit_event` natif (OCA `auditlog` exclu — antipattern, 45× dégradation perf).
- Archivage trimestriel Parquet S3 + WORM S3 Object Lock pour partitions froides.

**Conformité** :

- RGPD : anonymisation reversible sans DELETE physique, audit horodaté multi-pays (NFR-19, FR-73-76).
- France : délai coupure paramétrable défaut 15j calendaires (marge jurisprudentielle 8-15j).
- ARCEP / directive EECC : encadrement durée/résiliation, pas de coupure les jours interdits.
- **OQ-20 ouverte** : responsable unique vs co-responsabilité art. 26 (DPO opérateur).

**Calendriers multi-pays** :

- Lib `python-holidays` : FR métropole + DOM-TOM (Guadeloupe, Martinique, Saint-Martin, Guyane) + Mali + Sénégal.
- Surcharge manuelle possible côté admin (FR-27, CAP-9).

### Cross-Cutting Concerns Identified

| Concern | Implémentation cible | Risque / NFR mitigé |
|---|---|---|
| Audit immuable | `axia_audit_event` + `pg_partman` (mensuel) + WORM S3 | R4, FR-46-50, FR-62-68a |
| Sécurité secrets | OCA `keychain` + clé maître hors-base | NFR-14, NFR-25 |
| Observabilité J1 | Prometheus + alerting + dashboard Odoo + Loki | R6 (mode pionnier), NFR-20-24 |
| RBAC 7 rôles + délégation | Groupes Odoo + record rules + gating actions techniques | F7 / FR-77-83 |
| Idempotence | Outbox Odoo `(payload_hash, response)` | R10 (pas d'`Idempotency-Key`) |
| Réconciliation | Cron horaire full-scan + checksum + dedup GET | R2 (webhooks perdus) |
| Multi-tenant étanchéité | Multi-company natif + record rules par marque | FR-81 (4 backends étanches) |
| Sharding charge | Channels queue_job `root.sync.<company_id>:1` | NFR-9 (fairness) |
| Calendrier multi-pays | `python-holidays` + surcharge admin sans code | FR-27, CAP-9 |
| PPP Odoo-owned | Algo génération (OQ-22) + rotation gated + chiffrement | OQ-22, FR-11a/b/c, FR-78a, NFR-25 |
| Anti-replay webhook | UNIQUE INDEX `event_id` fenêtre 5 min + HMAC constant-time | NFR-15, NFR-16 |
| Reprise crash queue_job | Runbook + alerting `started_for > 30m` | R14, NFR-13 |

### Open Questions critiques (ADR Phase 0)

| OQ | Sujet | Impact architecture |
|---|---|---|
| **OQ-7** | Payloads réels POST customer / internet-service | Bloque la couche Mapper — POC tenant Splynx |
| **OQ-15a** | Reprise PPP legacy (bascule v1→v2) | Import passif, freeze baseline, audit `ppp.imported.legacy` |
| **OQ-20** | Statut juridique (resp. unique vs co-resp art. 26) | Multi-company vs multi-DB |
| **OQ-22** | Algorithme/format PPP généré Odoo | Charset, entropie, longueur, anti-collision |

---

## Starter Template Evaluation

### Primary Technology Domain

**Backend / data sync / intégration Odoo 18** — on part sur un déploiement on-premise Docker, PostgreSQL 15, de la queue async via OCA `queue_job`. Dans l'écosystème Odoo, il n'y a pas de starter CLI générique qui fait tout : la convention dominante, c'est un **repo monolithique Docker Compose** qui orchestre Odoo + les addons custom + les OCA vendorisés.

### Starter Options Considered

1. **`odoo scaffold`** (commande native) — squelette de module minimal, **retenu par module** mais pas pour l'orchestration projet.
2. **`OCA/cookiecutter-odoo-module`** — adapté modules OCA standalone (CI, runboat). **Non retenu** : on construit une suite intégrée, pas un module isolé à publier sur l'OCA.
3. **Odoo.sh template SaaS** — **exclu d'office** par décision actée (déploiement on-premise).
4. **Repo custom Docker Compose** (retenu) — orchestre Odoo, Postgres, Redis (queue_job), Prometheus, scripts pg_partman + S3.

### Selected Starter: Repo custom Docker Compose + `odoo scaffold` par module

**Pourquoi ce choix** :

- Les décisions de stack sont figées par le PRD v2 + l'addendum (decision log entries 3, 7, 13).
- Il n'existe pas d'alternative dans l'écosystème Odoo qui couvre à la fois multi-company + OCA connector + pg_partman + WORM S3.
- Ça évite la friction d'un starter SaaS (Odoo.sh) incompatible avec l'on-premise.
- `odoo scaffold` natif suffit pour bootstrapper chaque module ; l'orchestration passe par Docker Compose + des scripts custom.

**Initialization Commands** :

```bash
# Bootstrap repo (sprint 0)
git init && cp docker-compose.yml.example docker-compose.yml
docker compose build odoo
docker compose up -d postgres redis

# Bootstrap nouveau module
docker compose run --rm odoo odoo scaffold axia_<nom> /mnt/extra-addons/

# Pre-commit OCA-compliant
pip install pre-commit && pre-commit install
```

### Architectural Decisions Provided by the Stack

**Langage & runtime** :

- Python 3.11+ (Odoo 18 LTS requirement)
- ORM Odoo natif + Postgres 15

**Persistance** :

- PostgreSQL 15 (Odoo standard) + extension `pg_partman` (audit)
- S3 (compatible) avec Object Lock (WORM) pour archivage froid Parquet trimestriel

**Job queue** :

- OCA `queue_job` (branch 18.0) — worker dédié séparé de l'UI (NFR-12)
- Channels par société : `root.sync.<company_id>:1` (NFR-9 fairness)
- Redis backend optionnel mais recommandé (reprise crash NFR-13)

**Sécurité** :

- OCA `keychain` (branch 18.0) — secrets API + `Pass ppp`
- Clé maître via variable d'environnement (jamais en base — NFR-25)

**Connector pattern** :

- OCA `connector` (branch 18.0) — binding models, backend records, mappers
- Client HTTP : `httpx` custom (~200 LOC) — pas le SDK Splynx (R8)

**Build & déploiement** :

- Docker + Docker Compose (on-premise)
- Image Odoo 18 Community + addons vendored (third_party_addons/)

**Observabilité** :

- Prometheus client Python (exporter custom)
- Logs JSON structurés (compatible Loki)
- Dashboard santé connecteur dans Odoo (NFR-23, pas de Grafana externe)

**Convention OCA** :

- Manifest, structure, `.pre-commit-config.yaml` (black, isort, pylint-odoo)
- Pas de publication OCA upstream — suite intégrée custom

### Scaffold attendu (7 modules)

> **Note décision party-mode étape 3** : `axia_core` initialement envisagé a été **fusionné dans `axia_audit`**. Le mixin `axia.audited.mixin` et le helper `correlation_id` sont audit-centriques (consommés par 5 modules sur 7). Les helpers calendaires migrent dans `axia_billing_workflow` (seul consommateur réel : CAP-9). Pas d'umbrella manifest séparé.

```
splynx-odoo/                          # repo monolithique
├── docker-compose.yml                # services : odoo, postgres, redis, prometheus
├── docker-compose.override.yml       # dev only
├── Dockerfile                        # Odoo 18 base + deps Python (httpx, python-holidays…)
├── requirements.txt                  # versions pinnées Python
├── odoo.conf                         # workers, db_filter, queue_job channels
├── .env.example                      # secrets template (clé maître keychain hors-base)
├── addons/
│   ├── axia_splynx_connector/        # OCA connector + httpx client + mappers + outbox/inbox
│   ├── axia_audit/                   # axia_audit_event + pg_partman + WORM S3
│   │                                 # + axia.audited.mixin + axia.correlation
│   ├── axia_rbac/                    # 7 rôles + record rules + délégation partenaires + gating
│   ├── axia_ppp/                     # génération PPP Odoo-owned + rotation + chiffrement keychain
│   ├── axia_billing_workflow/        # CAP-7 à CAP-13 (impayés, scheduler, calendaires)
│   ├── axia_admin_params/            # paramètres admin sans code (CAP-14, FR-41-45)
│   └── axia_observability/           # exporter Prometheus + dashboard Odoo (NFR-20-24)
├── third_party_addons/               # vendoring OCA (submodule ou copie pinnée)
│   ├── connector/
│   ├── queue_job/
│   └── keychain/
├── scripts/
│   ├── pg_partman_init.sql           # création partitions axia_audit_event
│   ├── archive_to_parquet.py         # job trimestriel S3 + WORM
│   └── runbook/                      # asymétrie blocage, jobs `started` > 30m…
├── ci/
│   ├── .pre-commit-config.yaml       # OCA-compliant
│   └── github-actions/               # tests, lint, build image
└── docs/
    ├── architecture.md (ce doc)
    ├── adr/                          # ADR Phase 0 (OQ-7, OQ-15a, OQ-20, OQ-22)
    └── runbook/
```

### Graphe de dépendances modules

```
axia_audit                  ← (foundation : mixin + correlation_id + storage partitionné)
  ↑
  ├── axia_rbac             (audit attributions droits FR-83)
  ├── axia_splynx_connector (audit syncs + outbox)
  ├── axia_ppp              (audit ppp.generated/rotated/revealed/imported.legacy)
  ├── axia_admin_params     (audit paramètres modifiés)
  ├── axia_billing_workflow (audit décisions impayés + reports)
  └── axia_observability    (lit metrics depuis axia_audit_event)
```

### Open Question reportée au sprint 0

**OQ-6** — Audit maturité branches OCA 18.0 (`connector`, `queue_job`, `keychain`) au moment du démarrage : versions précises à figer dans `requirements.txt` et `docker-compose.yml`. Relecture trimestrielle (R5).

**Note** : l'initialisation effective du repo (commande `git init` + scaffold Docker Compose + premier `odoo scaffold axia_audit`) constitue la **première story d'implémentation** (Sprint 0).

---

## Core Architectural Decisions

### Decision Priority Analysis

**Critical Decisions — ADR Phase 0 (bloquantes Sprint 0)** :

| ADR | Sujet | OQ adressée |
|---|---|---|
| ADR-001 | Payloads Splynx & contrat Mapper | OQ-7 |
| ADR-002 | Reprise PPP legacy | OQ-15a |
| ADR-003 | Statut juridique propriétaire (multi-company vs multi-DB) | OQ-20 |
| ADR-004 | Algorithme & format PPP généré Odoo | OQ-22 |

**Important Decisions — Formalisations standard** :

| ADR | Sujet |
|---|---|
| ADR-005 | Schéma `axia_audit_event` & partitionnement |
| ADR-006 | Sécurité webhooks Splynx |
| ADR-007 | Authentification API Splynx |
| ADR-008 | Client HTTP Splynx (`httpx` custom) |
| ADR-009 | Fairness queue_job multi-tenant |
| ADR-010 | Observabilité J1 (mode pionnier) |

**Deferred Decisions (Sprint 0 ops ou post-MVP)** :

| OQ | Sujet | Différé jusqu'à |
|---|---|---|
| OQ-6 | Maturité OCA 18.0 | Sprint 0 (audit pré-pinning) |
| OQ-9 | Versions MySQL/MariaDB Splynx | Sprint 0 ops |
| OQ-12 | Maturité Sentry + queue_job | Sprint 0 |
| OQ-13 | Connecteur Splynx propriétaire Apps Store | Recherche PM |
| OQ-14 | Stabilité IP tenant Splynx (allow-list) | Contact Splynx |
| OQ-16 | Format export comptable | PM + opérateur |
| OQ-19 | Hébergement multi-pays | Projet infra séparé |
| OQ-21 | Marqueur ISP service (`is_isp_service` source) | PM + opérateur |

---

### Data Architecture

#### **ADR-005 — Schéma `axia_audit_event` & partitionnement**

**Ce qu'on a retenu** : une table custom partitionnée mensuellement — surtout pas OCA `auditlog` (l'addendum le confirme : c'est un antipattern qui dégrade les perfs d'un facteur 45). L'idée, c'est de construire une fondation d'audit légère, scalable, qui ne deviendra pas un goulot d'étranglement quand on atteindra 100k abonnés.

**Schéma** :

```sql
CREATE TABLE axia_audit_event (
    id BIGSERIAL,
    correlation_id UUID NOT NULL,
    event_type VARCHAR(64) NOT NULL,        -- customer.synced, ppp.rotated, etc.
    actor_user_id INTEGER REFERENCES res_users(id),
    target_model VARCHAR(64) NOT NULL,
    target_id BIGINT,
    company_id INTEGER REFERENCES res_company(id),
    payload_json JSONB,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);
```

**Indexes** :
- `(correlation_id)` — reconstitution chaîne événement E2E (NFR-24)
- `(target_model, target_id, created_at DESC)` — vue chronologique fiche client
- `(company_id, event_type, created_at DESC)` — agrégats observability + RGPD

**Politique rétention** :
- 12 mois chauds en PG (partitions mensuelles via `pg_partman`)
- 36-48 mois froids en Parquet S3 + Object Lock WORM (job trimestriel `scripts/archive_to_parquet.py`)
- 5 ans glissants total, purge automatique au-delà

**Mixin partagé** : `axia.audited.mixin` (dans `axia_audit`) expose `_audit(event_type, payload)` consommé par 5 modules sur 7.

#### **ADR-002 — Reprise PPP legacy (OQ-15a)**

**Ce qu'on a retenu** : un **import passif** à la bascule v1→v2, sans rotation forcée. En clair, on ne casse rien : on importe les PPP existants tels quels, on les gèle, et on laisse chaque opérateur décider quand migrer vers le nouveau format Odoo.

**Flux** :
1. Job `axia.ppp.import.legacy` lancé une fois par backend Sprint 1 (post-déploiement).
2. Pour chaque service Splynx existant : créer `axia.ppp.binding` avec `login` + `password` chiffré keychain + `source = 'legacy'` + `imported_at`.
3. Événement audit `ppp.imported.legacy` (1 par binding, correlation_id par batch).
4. **Freeze baseline** : snapshot chiffré complet des PPP legacy archivé S3 WORM (rollback de secours).
5. PPP legacy reste actif jusqu'à rotation manuelle volontaire (FR-11c).
6. Dashboard observability : compteur "PPP legacy actifs vs Odoo-generated" par backend.

**Acceptation Sprint 1** : 100 % services existants ont un binding, 0 rupture PPPoE, snapshot S3 WORM vérifié.

#### **ADR-003 — Multi-company vs multi-DB (OQ-20)**

**Ce qu'on a retenu** : on part en **multi-company natif par défaut**, mais avec un code totalement agnostique au mode. Comme ça, si le DPO impose du multi-DB plus tard, la bascule est triviale — aucun refactor applicatif.

**Contraintes architecture** :
- Aucun cross-company SQL hardcodé. Tous les filtres passent par record rules `company_id`.
- `BackendAdapter` instancié par société, jamais cross-société.
- Si DPO impose multi-DB (cloisonnement strict art. 26) : config `dbfilter` + 4 instances Odoo (ou 1 process Odoo + dbfilter), aucun refactor code applicatif.
- Décision DPO **doit être obtenue avant déploiement prod**, mais **ne bloque pas** le développement v1.

**Test acceptation Sprint 0** : pénétration cross-company (R13) — user marque X ne voit aucune donnée marque Y, ni en UI ni via JSON-RPC direct (100 % couverture).

---

### Authentication & Security

#### **ADR-004 — Algorithme & format PPP généré Odoo (OQ-22)**

**Ce qu'on a retenu** : un login structuré qui ne fuit pas d'information, et un mot de passe à haute entropie cryptographique. L'important, c'est que le login reste lisible pour le support technique, mais que le mot de passe soit blindé — chiffré en base, jamais loggé, jamais en clair.

**Login** :
- Format : `<marque-slug>-<base32_8chars>` (ex. `marque-a-k7m9p2qx`)
- `marque-slug` ∈ {`marque-a`, `marque-b`, `marque-c`, `marque-d`}
- Entropie : 40 bits (`secrets.token_bytes(5)` → base32 lowercase)
- **Pre-check anti-collision** (FR-11b) : `GET admin/services/internet?login=<candidat>` avant POST ; si trouvé → régénération + retry (max 3, alerte au-delà)
- Login en clair (identité technique, pas sensible)

**Password** :
- 24 caractères, charset `string.ascii_letters + string.digits` (62 chars)
- Génération : `''.join(secrets.choice(charset) for _ in range(24))` → ~143 bits d'entropie
- **Chiffré at-rest via OCA `keychain`** (FR-78a, NFR-25), jamais loggé, jamais en variable d'env plaintext
- **Révélation contrôlée** (FR-78a) : gated par rôle + audit `ppp.revealed` avant retour à l'UI

**Rotation (FR-11c)** :
- Gated par rôle (Contentieux + Administratif + Direction)
- Génère nouveau couple, propage Splynx, audit `ppp.rotated`
- Ancien password purgé du keychain à T+0

**Format pluggable** : `axia.ppp.generator` est un service Odoo (config admin) — algo modifiable sans redéploiement si Splynx impose contrainte future.

**Acceptation Sprint 1** : 1000 générations consécutives → 0 collision pre-check ; tests entropie passwords ; conformité charset Splynx vérifiée tenant test.

#### **ADR-006 — Sécurité webhooks Splynx**

**Ce qu'on a retenu** : un controller ultra-léger qui ne fait que vérifier et stocker, puis une outbox asynchrone qui fait le vrai boulot. Avec de l'anti-replay strict pour qu'un même événement ne soit jamais traité deux fois.

**Flux** :
1. Endpoint controller Odoo `/axia/webhook/<backend_slug>` ultra-léger (< 500 ms NFR-11)
2. Vérification HMAC-SHA1 (algo Splynx, pas SHA256) en `hmac.compare_digest` constant-time (NFR-15)
3. Insertion `axia.webhook.inbox` (table outbox/inbox dédiée)
4. Réponse HTTP 200 immédiate à Splynx
5. Job queue_job `process_webhook` consomme l'inbox de manière asynchrone (channel `root.webhook`)

**Anti-replay** :
- `UNIQUE INDEX (event_id, backend_id)` sur `axia.webhook.inbox` (NFR-16)
- Fenêtre rejet 5 min (timestamp drift toléré)

**Defense in depth** :
- IP allow-list au niveau reverse proxy (Nginx/Traefik), pas dans Odoo
- Secret HMAC par backend, stocké keychain (un secret par marque)
- HTTPS obligatoire TLS 1.2+ (NFR-17)

#### **ADR-007 — Authentification API Splynx**

**Ce qu'on a retenu** : 1 compte API dédié par backend, avec le scope le plus étroit possible, en token-based (pas de basic auth).

**En pratique** :
- 4 comptes API distincts (un par marque), créés dans chaque tenant Splynx
- Scope minimal : CRUD `customer` + CRUD `internet-service` + actions `block`/`unblock`/`terminate`
- Token-based (pas basic auth)
- Token stocké keychain, jamais loggé, jamais en variable d'env plaintext
- **Rotation token** : procédure manuelle documentée runbook (pas automatisée v1, hors scope)

---

### API & Communication Patterns

#### **ADR-001 — Contrat Mapper & POC payloads Splynx (OQ-7)**

**Ce qu'on a retenu** : un POC sur tenant Splynx dès le Sprint 0, couplé à une interface Mapper pluggable et versionnée. L'idée, c'est que chaque backend peut évoluer indépendamment — un mapper V1 pour Marque A, un V2 pour Marque B — sans casser les autres.

**Architecture Mapper** :

```python
# axia_splynx_connector/services/mappers/base.py
class BaseMapper(ABC):
    version: str
    @abstractmethod
    def to_splynx_customer(self, odoo_partner: ResPartner) -> dict: ...
    @abstractmethod
    def to_splynx_internet_service(self, odoo_service: AxiaService) -> dict: ...
    @abstractmethod
    def from_splynx_monitoring(self, payload: dict) -> dict: ...
```

- Implémentations concrètes versionnées : `MapperV1`, `MapperV2`…
- `BackendAdapter` (par marque) référence une version de mapper (config admin)
- Tests par **golden files JSON** (snapshot des payloads tenant test)
- Rolling upgrade payload possible sans rupture (un backend peut migrer V1→V2 indépendamment)

**POC Sprint 0** :
- Capturer payloads réels via mitmproxy sur tenant test des 4 backends
- 2 golden files par opération minimum : `POST customer`, `POST internet-service`, `PUT customer`, `PUT internet-service`, monitoring response
- Mettre à jour `field-mappings.md` avec champs réels + types observés

**Acceptation Sprint 0** : `MapperV1` opérationnel sur les 4 backends, golden files validés par dry-run.

#### **ADR-008 — Client HTTP Splynx (`httpx` custom)**

**Ce qu'on a retenu** : un wrapper minimal d'environ 200 lignes, sans dépendre du SDK Splynx (la licence du SDK est floue, risque R8). On garde le contrôle total : timeouts, retries, logging sécurisé, métriques Prometheus.

**Spécification** (`axia_splynx_connector/services/splynx_client.py`) :
- Base : `httpx.Client` avec connection pooling
- Timeout : 10 s (configurable par backend)
- Retry : exponentiel sur 5xx + 429 (3 essais, backoff 1s/2s/4s)
- Auth : token injecté depuis keychain (lazy load au premier appel)
- Logging : JSON structuré avec correlation_id, jamais le token, jamais le payload `Pass ppp`
- Metrics Prometheus : latence p95, taux erreur par endpoint, rate-limit hit (NFR-20)
- Tests : `respx` ou VCR.py pour mocker (pas de test live sur tenant en CI)

**Interface** (extrait) :

```python
class SplynxClient:
    def __init__(self, backend: AxiaSplynxBackend): ...
    def get_customer(self, splynx_id: int) -> dict: ...
    def create_customer(self, payload: dict) -> int: ...
    def update_customer(self, splynx_id: int, payload: dict) -> None: ...
    def block_service(self, service_id: int) -> None: ...
    def unblock_service(self, service_id: int) -> None: ...
    # + variantes internet-service, monitoring read…
```

---

### Infrastructure & Deployment

#### **ADR-009 — Fairness queue_job multi-tenant**

**Ce qu'on a retenu** : des channels dédiés par société, pour que chaque marque ait sa propre file et qu'aucune ne monopolise les workers. On sépare aussi la charge sync, webhook et réconciliation pour éviter les interférences.

**Configuration** (`odoo.conf`) :

```ini
[queue_job]
channels = root:1, \
           root.sync.marque-a:1, root.sync.marque-b:1, \
           root.sync.marque-c:1, root.sync.marque-d:1, \
           root.webhook:5, \
           root.reconcile:1, \
           root.legacy_import:1
```

**Pourquoi cette config** :
- Concurrency 1 par marque sur sync (anti rate-limit Splynx — ajustable post-POC R3)
- Webhook channel concurrency 5 (faible latence requise, NFR-11)
- Réconciliation cron channel séparé basse priorité (n'écrase pas la sync temps réel)
- Import legacy channel dédié (one-shot, n'interfère pas Sprint 1+)

**Backend** : Redis recommandé pour persistance jobs (reprise crash NFR-13).

**Runbook critique** (R14) : alerting sur jobs `started_for > 30m` → procédure requeue manuelle documentée.

#### **ADR-010 — Observabilité J1 (mode pionnier R6)**

**Ce qu'on a retenu** : Prometheus + Alertmanager pour les métriques et l'alerte, un dashboard directement dans Odoo pour la visibilité quotidienne, Loki pour les logs, et Sentry en option pour les crashs. Pas de Grafana externe en v1 — on réduit la surface d'attaque et la charge ops.

**Métriques exposées** (NFR-20) — endpoint `/axia/metrics` :
- `axia_queue_backlog{channel="..."}` (gauge)
- `axia_splynx_request_duration_seconds{backend, endpoint}` (histogram)
- `axia_splynx_error_rate_24h{backend}` (gauge)
- `axia_sync_last_ok_seconds_ago{backend}` (gauge)
- `axia_ppp_legacy_active{backend}` (gauge — décroissance attendue)
- `axia_webhook_processing_duration_seconds{backend, event_type}` (histogram)

**Alerting** (NFR-21) — règles Prometheus :
- `axia_queue_backlog > 100 for 5m` → page ops
- `axia_splynx_error_rate_24h > 0.05` → page ops
- `axia_sync_last_ok_seconds_ago > 3600` → page ops
- `axia_queue_job_started_too_long{started_for_seconds > 1800}` → page ops (R14)
- Retard réactivation > 10 min après paiement (FR-32, NFR-3) → page ops

**Logs** (NFR-22) :
- JSON structuré (timestamp ISO, level, correlation_id, customer_id, company_id, event)
- Stream vers Loki (compat docker-compose)
- Rétention 30 jours hot, archive S3 selon politique audit

**Dashboard Odoo natif** (NFR-23) :
- Vue kanban "Santé connecteur" par backend
- Refresh 30 s
- Pas de Grafana externe v1 (réduction surface attaque + dépendance ops)

**Correlation_id** (NFR-24) :
- Helper `axia.correlation` (dans `axia_audit`) : génère UUID v4 au point d'entrée (webhook, cron, UI action)
- Propagé end-to-end via context manager + `_audit()` mixin
- Tous les logs JSON le contiennent

**Sentry** (OQ-12) : hook `on_job_failed` queue_job pour erreurs non-récupérables, à valider Sprint 0.

---

### Decision Impact Analysis

**Implementation Sequence (ordre suggéré Sprint 0+1)** :

1. **Sprint 0** — Scaffold repo Docker Compose, `axia_audit` (ADR-005), POC payloads Splynx (ADR-001), pinning OCA (OQ-6), test pénétration cross-company (ADR-003).
2. **Sprint 1** — `axia_splynx_connector` (ADR-001 + ADR-008), `axia_rbac` (configuration 7 rôles), `axia_ppp` (ADR-004), `axia_observability` (ADR-010 base).
3. **Sprint 2** — `axia_billing_workflow` (impayés + calendaires), `axia_admin_params`, import legacy PPP (ADR-002), runbook complet (R7, R14).

**Cross-Component Dependencies** :

- ADR-001 (Mapper) ↔ ADR-008 (Client) : interface stable `dict → HTTP`
- ADR-004 (PPP algo) ↔ ADR-002 (legacy) : les PPP legacy n'utilisent PAS l'algo Odoo (`source = 'legacy'`), uniquement les nouveaux
- ADR-006 (webhooks) ↔ ADR-009 (channels) : webhooks dans channel dédié `root.webhook`, jamais dans channel sync
- ADR-003 (multi-company) ↔ ADR-009 (channels) : si bascule multi-DB, un seul channel par DB
- ADR-005 (audit) ↔ ADR-010 (observability) : Prometheus exporter lit des agrégats `axia_audit_event`, pas de double-écriture metrics
- ADR-001 (Mapper) ↔ ADR-005 (audit) : chaque sync écrit événement `customer.synced` ou `service.synced` avec payload mapper en `payload_json`

---

## Implementation Patterns & Consistency Rules

### Pattern Categories Defined

**Critical conflict points identifiés** : 12 zones spécifiques AXIA où des agents IA pourraient diverger sans consigne. Les conventions Odoo standard + OCA (pylint-odoo, black, isort, manifest format, snake_case modèles, etc.) sont **considérées acquises** et ne sont pas redocumentées ici.

### Naming Patterns

#### Convention `event_type` (audit)

Format : `<domain>.<action>[.<modifier>]`, lowercase, dot-separated, max 64 chars.

**Exemples canoniques** :

| Domaine | Events |
|---|---|
| customer | `customer.synced`, `customer.sync.failed`, `customer.created`, `customer.terminated` |
| ppp | `ppp.generated`, `ppp.rotated`, `ppp.revealed`, `ppp.imported.legacy` |
| suspension | `suspension.executed`, `suspension.postponed`, `suspension.cancelled` |
| reactivation | `reactivation.executed`, `reactivation.failed` |
| role | `role.granted`, `role.revoked`, `partner.delegated` |
| webhook | `webhook.received`, `webhook.duplicate`, `webhook.invalid_hmac` |
| mapper | `mapper.upgraded` |

**Règles** :
- Lowercase strict, dot-separated, pas de camelCase, pas d'underscores dans le nom (sauf si valeur métier le justifie : `ppp.imported.legacy`).
- Inventaire central obligatoire : `axia_audit/data/event_types.csv` (référence).
- Test au boot : raise `AxiaAuditUnknownEventType` si un `_audit(event_type)` utilise un event non déclaré.

#### Channels queue_job

Format : `root.<purpose>[.<scope>]:<concurrency>`.

| Channel | Usage | Concurrency |
|---|---|---|
| `root.sync.marque-a` | Sync Marque A | 1 (anti rate-limit) |
| `root.sync.marque-b` | Sync Marque B | 1 |
| `root.sync.marque-c` | Sync Marque C | 1 |
| `root.sync.marque-d` | Sync Marque D | 1 |
| `root.webhook` | Webhooks entrants (toutes marques) | 5 (faible latence) |
| `root.reconcile` | Cron horaire de réconciliation | 1 (basse prio) |
| `root.legacy_import` | Import PPP legacy Sprint 1 | 1 (one-shot) |

Pas d'autre racine. Pas de namespace `axia.*` (convention OCA).

#### Bindings OCA `connector`

Format : `<module>.binding.<external_system>.<entity>`.

| Binding | Modèle |
|---|---|
| `axia.binding.splynx.customer` | Client |
| `axia.binding.splynx.internet_service` | Service Internet |
| `axia.binding.splynx.ppp` | Identifiants PPP |

Backend record : `axia.backend.splynx` (un record par marque, indexé par `company_id`).

#### Actions techniques

Format : `action_<verb>[_<context>]` (méthode Python sur binding).

Exemples : `action_suspend`, `action_reactivate`, `action_terminate`, `action_rotate_ppp`, `action_reveal_ppp`.

**Décorateurs obligatoires** :
- `@axia.requires_role('contentieux,administratif,direction')` (custom decorator dans `axia_rbac`) — gating server-side au-delà du gating UI.
- `@queue_job.job(default_channel='root.sync.<company>')` si appel Splynx (asynchrone obligatoire, jamais bloquer l'UI).

#### Mapper versioning

- Format : `Mapper<MAJOR>` (ex. `MapperV1`, `MapperV2`).
- Pas de date dans le nom.
- Bump majeur uniquement sur changement breaking payload Splynx.
- Fichier : `axia_splynx_connector/services/mappers/v<N>.py`. Abstract dans `base.py`.

### Structure Patterns

#### Organisation tests par module

```
axia_<module>/tests/
├── __init__.py
├── common.py                  # fixtures partagées
├── unit/                      # pas de DB
│   └── test_<unit>.py
├── integration/               # avec DB + transactions
│   └── test_<feature>.py
└── golden/                    # snapshots JSON (Mapper, payloads, audit)
    └── splynx_post_customer_v1.json
```

- Runner : `pytest` + `pytest-odoo` (recommandé), pas le test runner natif Odoo.
- CI : `--test-tags /axia_<module>` par module pour isolation.

#### XML data files par module

| Fichier | Contenu |
|---|---|
| `security/ir.model.access.csv` | ACL standard Odoo |
| `security/security.xml` | Groupes + record rules |
| `data/<entity>.xml` | Data techniques (event_types, channels…) |
| `demo/<entity>.xml` | Données démo (`noupdate="1"`) |

Pas de mélange security + data dans un même fichier.

#### Manifest deps

- `depends` exhaustif explicite (jamais d'imports indirects).
- Ordre canonique : `base` → modules Odoo standard → modules OCA → modules `axia_*` internes.
- Pas de glob d'imports.

### Format Patterns

#### Structure `payload_json` audit

Schéma standard (3 clés obligatoires + extension libre par `event_type`) :

```json
{
  "actor": {
    "user_id": 42,
    "user_login": "user@example.com",
    "company_id": 1
  },
  "target": {
    "model": "axia.service",
    "id": 12345,
    "external_id": 67890
  },
  "delta": {
    "field_name": {"before": "...", "after": "..."}
  },
  "...": "extensions libres par event_type"
}
```

- Validation côté `axia.audited.mixin._audit()` : raise en dev, log warning en prod si clés manquantes.
- `delta` peut être vide (événements non-mutateurs : révélation, lecture).
- Extensions libres documentées dans `event_types.csv` (colonne `payload_schema`).

#### Format logs JSON (stdout, compat Loki)

```json
{
  "ts": "2026-06-10T14:32:18.123Z",
  "level": "INFO",
  "logger": "axia.splynx.client",
  "msg": "splynx_request",
  "correlation_id": "...",
  "company_id": 1,
  "backend": "marque-a",
  "endpoint": "POST admin/customers/customer",
  "duration_ms": 142,
  "status": 200
}
```

- **Jamais** de `Pass ppp`, token API, payload complet (champs sélectionnés uniquement).
- Helper `axia.logger` (dans `axia_audit`) wrappe `logging` avec injection auto correlation_id + redaction secrets.

### Communication Patterns

#### Propagation `correlation_id`

Généré au point d'entrée (controller webhook, cron, UI button), propagé via context manager.

```python
with self.env['axia.correlation'].context(correlation_id):
    self.env['axia.audited.mixin']._audit('customer.synced', payload)
    self.env['axia.splynx.binding']._sync_to_splynx(customer)
```

- Stockage : `contextvars.ContextVar` (propagation auto à travers la stack Python).
- Propagation queue_job : via `metadata['correlation_id']` (lu au début du job, ré-injecté dans context).
- Format : UUID v4 string canonique 36 chars.

#### Naming des channels queue_job

Cf. section Naming Patterns.

### Process Patterns

#### Gestion d'erreurs queue_job

| Type | Action | Conséquence |
|---|---|---|
| **Retryable** (timeout, 5xx, 429) | `raise RetryableJobError(msg, seconds=N)` | Retry exponentiel auto |
| **Permanent** (4xx hors 429, validation Odoo, contradiction métier) | `raise FailedJobError(msg)` | Job état `failed` + alert Prometheus |
| **Inattendu** (bug) | `raise` brut | Job état `failed` + remontée Sentry (OQ-12) |

**Bannis** :
- `try/except Exception: pass` (silencer un bug)
- `try/except: ...` (catch nu)
- Ré-essai manuel infini sans `RetryableJobError`

### Enforcement Guidelines

**Tous les agents IA DOIVENT** :

1. Déclarer tout nouvel `event_type` dans `axia_audit/data/event_types.csv` **avant** de l'utiliser dans un `_audit()`.
2. Wrapper tout point d'entrée (controller, cron, UI button) avec `axia.correlation.context()` qui génère ou propage le UUID.
3. Utiliser `axia.logger` (jamais `logging.getLogger` direct) pour bénéficier de l'injection correlation_id + redaction.
4. Décorer toute action technique avec `@axia.requires_role(...)` + `@queue_job.job(...)` si appel Splynx.
5. Référencer un `event_type` valide dans `_audit(event_type, payload)` ; le mixin valide le schéma `payload`.
6. Écrire des tests `unit/` (sans DB) ET `integration/` (avec DB) pour toute nouvelle feature.
7. Ne **jamais** logger `Pass ppp`, token API, ou payload sensible complet.

**Vérification** :

| Pattern | Méthode de vérif |
|---|---|
| `event_type` déclaré | Boot test : raise si event_type non déclaré au `_audit()` call |
| `correlation_id` propagé | Test d'intégration E2E + grep logs |
| Schéma payload audit | Validation pydantic/jsonschema dans `_audit()` |
| Channels nommage | Pre-commit hook qui parse `__manifest__.py` + `odoo.conf` |
| `@requires_role` sur actions | Pylint custom rule (à écrire `axia_rbac/pylint_plugin.py`) |
| Pas de `try/except Exception: pass` | `pylint-odoo` enforce |
| Pas de secret dans logs | CI : grep `Pass ppp` / `token` dans output |
| Manifest deps explicites | `pylint-odoo` rule `missing-manifest-dependency` |

### Anti-Patterns explicitement bannis

- `_inherit = 'base'` pour patcher transversalement (bypass dépendances explicites)
- `self.env.cr.execute(...)` brut sans justification de perf commentée
- OCA `auditlog` (R4, antipattern documenté addendum — 45× dégradation perf)
- `try/except Exception: pass` (cf. process patterns)
- `print()` ou logs non structurés
- Hardcoded `company_id = 1` (toujours `self.env.company.id`)
- Imports relatifs cross-modules (toujours absolu)
- `sudo()` sans commentaire de justification
- INSERT direct dans `axia_audit_event` (toujours via mixin `_audit()`)
- Création de bindings sans backend explicite (`axia.backend.splynx` lookup)

---

## Project Structure & Boundaries

### Complete Project Directory Structure

```
splynx-odoo/
├── README.md
├── docker-compose.yml
├── docker-compose.override.yml          # dev only
├── Dockerfile
├── requirements.txt                     # versions pinnées Python
├── odoo.conf                            # workers, db_filter, queue_job channels
├── .env.example
├── .gitignore
├── .pre-commit-config.yaml              # OCA-compliant
├── pyproject.toml                       # pytest, black, isort config
│
├── addons/
│   │
│   ├── axia_audit/                      # FONDATION : storage + mixin + correlation
│   │   ├── __manifest__.py              # depends: base, mail
│   │   ├── models/
│   │   │   ├── axia_audit_event.py      # table partitionnée
│   │   │   ├── axia_audited_mixin.py    # mixin _audit() partagé
│   │   │   └── axia_correlation.py      # contextvars wrapper
│   │   ├── services/
│   │   │   ├── logger.py                # axia.logger (JSON + redaction)
│   │   │   └── archiver_s3.py           # job trimestriel Parquet S3 WORM
│   │   ├── data/
│   │   │   ├── event_types.csv          # inventaire central event_type
│   │   │   ├── ir_cron.xml              # cron archivage trimestriel
│   │   │   └── pg_partman_setup.xml
│   │   ├── security/
│   │   ├── views/
│   │   ├── wizards/export_audit_period.py   # export RGPD période/société
│   │   └── tests/{common.py, unit/, integration/}
│   │
│   ├── axia_rbac/                       # 7 rôles + record rules + délégation
│   │   ├── __manifest__.py              # depends: base, axia_audit
│   │   ├── models/{res_users.py, axia_partner_delegation.py}
│   │   ├── decorators/requires_role.py  # @axia.requires_role(...)
│   │   ├── security/
│   │   │   ├── axia_security.xml        # 7 groupes
│   │   │   └── axia_record_rules.xml    # cloisonnement company_id + marque
│   │   ├── wizards/delegate_partner_wizard.py
│   │   ├── pylint_plugin.py             # custom rule @requires_role
│   │   └── tests/integration/test_cross_company_penetration.py    # R13
│   │
│   ├── axia_splynx_connector/           # OCA connector + httpx + mappers + outbox
│   │   ├── __manifest__.py              # depends: connector, queue_job, keychain,
│   │   │                                # axia_audit, axia_rbac
│   │   ├── models/
│   │   │   ├── axia_backend_splynx.py   # 1 par marque
│   │   │   ├── axia_binding_customer.py
│   │   │   ├── axia_binding_internet_service.py
│   │   │   ├── axia_webhook_inbox.py    # outbox/inbox
│   │   │   └── axia_sync_outbox.py      # (payload_hash, response)
│   │   ├── services/
│   │   │   ├── splynx_client.py         # httpx wrapper ~200 LOC
│   │   │   └── mappers/{base.py, v1.py}
│   │   ├── controllers/webhook_splynx.py
│   │   ├── jobs/
│   │   │   ├── sync_customer.py
│   │   │   ├── sync_internet_service.py
│   │   │   ├── process_webhook.py
│   │   │   └── reconcile_cron.py        # cron horaire R2
│   │   ├── data/{event_types.csv, queue_job_channels.xml, ir_cron_reconcile.xml}
│   │   └── tests/{common.py, unit/, integration/, golden/}
│   │       └── golden/                  # snapshots payloads Splynx
│   │
│   ├── axia_ppp/                        # génération + rotation + chiffrement
│   │   ├── __manifest__.py              # depends: keychain, axia_audit, axia_rbac,
│   │   │                                # axia_splynx_connector
│   │   ├── models/axia_ppp_binding.py
│   │   ├── services/{ppp_generator.py, ppp_keychain.py}
│   │   ├── wizards/{rotate_ppp_wizard.py, reveal_ppp_wizard.py}
│   │   ├── jobs/import_legacy_ppp.py    # ADR-002 one-shot
│   │   └── tests/
│   │
│   ├── axia_billing_workflow/           # CAP-7 à CAP-13 (impayés + calendaires)
│   │   ├── __manifest__.py              # depends: account, axia_audit, axia_rbac,
│   │   │                                # axia_splynx_connector, axia_admin_params
│   │   ├── models/{account_move.py, axia_suspension_decision.py, res_partner.py}
│   │   ├── services/{suspension_engine.py, calendar_holidays.py, timezone_resolver.py}
│   │   ├── jobs/{detect_overdue.py, execute_suspension.py, execute_reactivation.py}
│   │   ├── data/{event_types.csv, ir_cron_overdue.xml}
│   │   └── tests/
│   │
│   ├── axia_admin_params/               # CAP-14 paramètres admin sans code
│   │   ├── __manifest__.py              # depends: base, axia_audit
│   │   ├── models/axia_admin_settings.py    # res.config.settings extension
│   │   ├── data/{event_types.csv, default_params.xml}
│   │   └── tests/
│   │
│   └── axia_observability/              # NFR-20-24 Prometheus + dashboard
│       ├── __manifest__.py              # depends: axia_audit, axia_splynx_connector
│       ├── controllers/metrics.py       # GET /axia/metrics
│       ├── services/{prometheus_collector.py, sentry_hook.py}
│       ├── views/connector_health_kanban.xml    # dashboard Odoo (NFR-23)
│       └── data/{prometheus_rules.yml.example, alertmanager_rules.yml.example}
│
├── third_party_addons/                  # OCA vendoré (branch 18.0)
│   ├── connector/
│   ├── queue_job/
│   └── keychain/
│
├── scripts/
│   ├── pg_partman_init.sql
│   ├── pg_partman_maintain.sh
│   ├── archive_to_parquet.py            # job trimestriel S3 + WORM
│   ├── pen_test_cross_company.py        # R13 (ADR-003 acceptation)
│   └── runbook/
│       ├── 01_asymetrie_blocage_splynx.md   # R7
│       ├── 02_queue_job_started_30m.md      # R14
│       ├── 03_rotate_splynx_token.md        # ADR-007
│       ├── 04_legacy_ppp_import.md          # ADR-002
│       └── 05_failover_multi_db.md          # ADR-003 bascule option C
│
├── ci/
│   ├── github-actions/{ci.yml, release.yml}
│   └── pytest-odoo.ini
│
├── deploy/
│   ├── prometheus/prometheus.yml.example
│   ├── alertmanager/alertmanager.yml.example
│   └── nginx/nginx.conf.example         # IP allow-list webhook (ADR-006)
│
└── docs/
    ├── architecture.md                  # CE DOCUMENT
    ├── adr/
    │   ├── ADR-001-mapper-payloads-splynx.md
    │   ├── ADR-002-legacy-ppp-import.md
    │   ├── ADR-003-multi-company.md
    │   ├── ADR-004-ppp-generation.md
    │   ├── ADR-005-audit-schema.md
    │   ├── ADR-006-webhook-security.md
    │   ├── ADR-007-splynx-api-auth.md
    │   ├── ADR-008-httpx-client.md
    │   ├── ADR-009-queue-job-channels.md
    │   └── ADR-010-observability.md
    ├── runbook/                         # copie de scripts/runbook
    └── handoff-implementation.md        # étape 8
```

### Requirements to Structure Mapping (Capability → Module)

| Capability | FRs principaux | Module(s) primaire(s) | Modules consommateurs |
|---|---|---|---|
| CAP-1 (création client) | FR-1, FR-1a/b/c/d/e | `axia_splynx_connector` | `axia_audit`, `axia_rbac` |
| CAP-2 (déduplication) | FR-4 à FR-6 | `axia_splynx_connector` | — |
| CAP-3 (sync incrémentale) | FR-7, FR-8, FR-53 | `axia_splynx_connector` | — |
| CAP-4 (attributs techniques + PPP) | FR-9 à FR-11c | `axia_ppp` + `axia_splynx_connector` | `axia_rbac` |
| CAP-5 (statuts contractuels) | FR-15 à FR-17 | `axia_splynx_connector` | `axia_billing_workflow` |
| CAP-6 (monitoring) | FR-12 à FR-14 | `axia_splynx_connector` | `axia_observability` |
| CAP-7 (impayés détection) | FR-22, FR-23, FR-38 | `axia_billing_workflow` | `axia_admin_params` |
| CAP-8 (modes suspension) | FR-24 à FR-26 | `axia_billing_workflow` | `axia_admin_params` |
| CAP-9 (calendaires multi-pays) | FR-27 à FR-29 | `axia_billing_workflow` | `axia_admin_params` |
| CAP-10 (report manuel) | FR-30, FR-31 | `axia_billing_workflow` | — |
| CAP-11 (réactivation 24/7) | FR-32 à FR-34 | `axia_billing_workflow` | `axia_splynx_connector` |
| CAP-12 (actions techniques) | FR-19, FR-21, FR-79 | `axia_splynx_connector` | `axia_rbac` |
| CAP-13 (workflow impayés E2E) | FR-35 à FR-40 | `axia_billing_workflow` | toutes |
| CAP-14 (admin sans code) | FR-41 à FR-45 | `axia_admin_params` | `axia_audit` |
| CAP-15 (audit 5 ans) | FR-46 à FR-50, FR-62 à FR-68a | `axia_audit` | toutes |
| F7 (RBAC 7 rôles) | FR-77 à FR-83 | `axia_rbac` | toutes |
| Multi-sociétés | FR-57 à FR-61 | natif Odoo + record rules `axia_rbac` | toutes |
| RGPD | FR-73 à FR-76 | `axia_audit` (wizards) + `axia_rbac` | — |

### Architectural Boundaries

#### Communication intra-Odoo (entre modules)

- **Mixin pattern** : `axia.audited.mixin` (dans `axia_audit`) hérité par tous les modèles écrivant des événements. Pas d'INSERT direct dans `axia_audit_event`.
- **Decorator pattern** : `@axia.requires_role(...)` (dans `axia_rbac`) sur toute action technique. Importé par tous les modules consommateurs.
- **Service pattern** : services métier (`axia.ppp.generator`, `axia.calendar.holidays`, `axia.splynx.client`) exposés via lookup ORM `self.env['axia.xxx']`.
- **Pas de cross-module SQL direct** : toujours passer par l'ORM Odoo (record rules respectées automatiquement).
- **Pas de cross-module Python import au-delà de l'API publique** : un module n'importe que les classes documentées dans le `__init__.py` du module cible.

#### Frontières externes

Tu trouveras ci-dessous la cartographie de toutes les connexions externes, protocole par protocole :

| Système externe | Module Odoo proxy | Protocole | Auth | Direction |
|---|---|---|---|---|
| **Splynx API** (4 backends) | `axia_splynx_connector/services/splynx_client.py` | HTTPS REST | Token (keychain) | Sortant Odoo → Splynx |
| **Splynx Webhooks** | `axia_splynx_connector/controllers/webhook_splynx.py` | HTTPS POST | HMAC-SHA1 | Entrant Splynx → Odoo |
| **S3 (audit froid)** | `axia_audit/services/archiver_s3.py` | HTTPS S3 API | IAM key (keychain) | Sortant Odoo → S3 (WORM) |
| **Redis (queue_job)** | OCA `queue_job` natif | Redis protocol | Password (env) | Intra-cluster |
| **PostgreSQL** | Odoo ORM natif | Postgres protocol | Password (env) | Intra-cluster |
| **Prometheus** | `axia_observability/controllers/metrics.py` | HTTPS GET `/axia/metrics` | IP allow-list | Entrant Prometheus → Odoo |
| **Sentry** (optionnel) | `axia_observability/services/sentry_hook.py` | HTTPS POST | DSN (keychain) | Sortant Odoo → Sentry |

#### Frontières de données

- **Sources de vérité** : Odoo pour commercial/PPP/statuts contractuels, Splynx pour monitoring temps réel (cf. `sync-ownership.md`).
- **Stockage chiffré at-rest** : `Pass ppp` + tokens API + DSN Sentry dans OCA `keychain` (clé maître via env var, NFR-25).
- **Stockage WORM** : partitions froides audit (12+ mois) en Parquet S3 Object Lock (immutable).
- **Stockage transient** : queue_job dans Redis (perte tolérée si reprise dans 5 min — NFR-13).
- **Isolation par société** : record rules `company_id` sur tous les modèles `axia.*`. Test pénétration cross-company en CI.

### Integration Points (data flows)

#### Flow 1 — Création client (CAP-1)

```
Odoo UI → res.partner.create(values)
       → @api.model trigger → axia_splynx_connector check eligibility
       → axia.binding.splynx.customer create (sync=False initial)
       → queue_job.delay sync_customer (channel root.sync.<company>)
       → splynx_client.create_customer(payload from MapperV1)
       → on success : binding.external_id = splynx_id, audit customer.synced
       → on retryable : RetryableJobError → retry exponentiel
       → on failed : FailedJobError → state=failed, alert Prometheus
```

#### Flow 2 — Webhook Splynx (FR-12, CAP-6 + CAP-13)

```
Splynx → POST /axia/webhook/marque-a (HMAC header)
       → controller webhook_splynx :
         - vérifie HMAC constant-time
         - upsert axia.webhook.inbox (UNIQUE event_id)
         - return 200 immédiat (< 500 ms NFR-11)
       → queue_job.delay process_webhook (channel root.webhook, concurrency 5)
       → dispatch par event_type : customer.updated → sync inverse,
                                   payment.received → trigger reactivation
       → audit webhook.received (ou webhook.duplicate si déjà vu)
```

#### Flow 3 — Workflow impayé (CAP-13)

```
account.move passe en state='late' (cron Odoo natif accounting)
       → axia_billing_workflow.detect_overdue (cron 15min NFR-1) :
         - lit axia_admin_settings (mode, délai, calendaires)
         - applique calendar_holidays.is_blocked(date, country)
         - applique cutoff_postponed_until check
         - décision : suspend now / postpone to D / skip
       → si suspend : axia.suspension.decision create
                    → queue_job.delay execute_suspension (channel root.sync.<company>)
                    → splynx_client.block_service(splynx_service_id)
                    → audit suspension.executed
       → si paiement reçu (event account.payment.create) :
                    → trigger execute_reactivation (24/7)
                    → splynx_client.unblock_service
                    → audit reactivation.executed
```

#### Flow 4 — Rotation PPP manuelle (FR-11c)

```
UI button "Régénérer PPP" (gated par @axia.requires_role)
       → wizard rotate_ppp_wizard :
         - axia.ppp.generator.generate(backend) → (login, password)
         - pre-check : splynx_client.find_service_by_login(login)
         - if collision : régénère (max 3 retries, audit failure si max atteint)
         - update axia.ppp.binding (password chiffré keychain)
         - queue_job.delay update_splynx_ppp
       → splynx_client.update_internet_service(payload with new ppp)
       → audit ppp.rotated (delta : old_login → new_login, password redacted)
       → ancien password purgé keychain à T+0
```

#### Flow 5 — Audit + observabilité

```
Toute opération mutateur dans Odoo :
       → axia.audited.mixin._audit(event_type, payload)
       → INSERT axia_audit_event (partition courante)
       → axia.logger.info(event, ...) → stdout JSON → Loki

Prometheus :
       → GET /axia/metrics (chaque 15s)
       → prometheus_collector lit agrégats sur axia_audit_event
                                + queue_job counts + httpx metrics
       → expose au format Prometheus standard

Alertmanager :
       → évalue rules (backlog, error rate, sync stale)
       → notifie ops (Slack/Mattermost/email)
```

### Development Workflow Integration

- **Dev local** : `docker compose up` → Odoo + Postgres + Redis + Prometheus accessibles localhost.
- **Tests** : `docker compose run --rm odoo pytest /mnt/extra-addons/axia_<module>/tests/` (par module isolé).
- **Pre-commit hooks** : black, isort, pylint-odoo, custom rule `@requires_role`.
- **CI GitHub Actions** : lint + tests par module en parallèle + build image Docker sur tag.
- **Déploiement** : push image sur registry privé, déploiement on-premise via Docker Compose ou Swarm.

### Build & Deployment Structure

- **Image Docker** : Odoo 18 Community base + Python deps (`requirements.txt`) + addons custom (`addons/`) + OCA vendored (`third_party_addons/`).
- **Volumes** : `odoo-data` (filestore), `postgres-data` (DB), `redis-data` (queue persistence).
- **Reverse proxy** : Nginx en front (TLS termination + IP allow-list webhook ADR-006).
- **Scaling vertical** v1 (workers Odoo + dedicated queue_job worker NFR-12) ; scaling horizontal possible v2 (multi-instance, sticky sessions, shared Redis).
