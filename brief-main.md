---
title: Brief Handoff Développement — Partie Commune
project: AXIA ISP Management Suite
audience: Équipe développement externe (4-5 devs, AI-native)
status: Ready for handoff
date: 2026-06-11
author: Winston (System Architect) + Paige (Technical Writer)
version: 3.0
---

# Brief Handoff — Partie Commune (AXIA ISP Management Suite)

> **À lire en premier, avant le brief du chantier en cours.** Tu trouveras ici les conventions, la stack, l'onboarding et la procédure de travail qui s'appliquent à **tous** les chantiers. Tu le reçois **en même temps** que le brief du chantier sur lequel tu démarres (Chantier 0, 1, 2, 3 ou 4). Bref, c'est ton point de repère — garde-le sous le coude tout au long du projet.

## Sommaire

1. [Package documentaire reçu](#1-package-documentaire-reçu)
2. [Vue d'ensemble projet](#2-vue-densemble-projet)
3. [Stack technique imposée](#3-stack-technique-imposée)
4. [Onboarding développeur](#4-onboarding-développeur)
5. [Conventions transverses & naming load-bearing](#5-conventions-transverses--naming-load-bearing)
6. [Definition of Done](#6-definition-of-done)
7. [Règles de déviation architecture (ADR-amendment)](#7-règles-de-déviation-architecture-adr-amendment)
8. [Ressources & contacts](#8-ressources--contacts)

**Annexe** — Glossaire express

---

## 1. Package documentaire reçu

Avec ce brief, tu reçois les documents suivants. **Mis ensemble, ils forment ta référence complète** — aucun autre document interne ne te sera transmis sauf demande spécifique pour un blocage. L'idée, c'est que tu aies tout ce qu'il faut sans être noyé sous une avalanche de paperasse interne.

| Document | Rôle | Statut |
|---|---|---|
| `brief-axia-isp-commun.md` (ce document) | Conventions transverses, stack, onboarding, DoD, ADR-amendment, ressources | À lire **en premier** |
| `brief-axia-isp-chantier-N.md` | Scope + stories + acceptance + tests d'acceptation **du chantier en cours uniquement** | À lire après ce document |
| `architecture.md` | Architecture canonique du projet : 10 ADR figés, scaffold des modules, patterns d'implémentation, frontières de données, flows d'intégration | Référence permanente |
| `SPEC.md` (+ companions) | Contrat sémantique : capacités CAP-1 à CAP-15, glossaire métier, matrice d'ownership des champs, états de service, paramètres admin | Référence permanente |

**Companions SPEC** (dossier `specs/spec-axia-isp/`) :
- `admin-parameters.md` — liste des paramètres métier éditables sans code
- `field-mappings.md` — matrice de correspondance champs Odoo ↔ Splynx (sera affinée par le POC payloads en Chantier 0)
- `service-states.md` — machine d'états des statuts contractuels
- `sync-ownership.md` — matrice de propriété par champ (Odoo-owned vs Splynx-owned)
- `glossary.md` — glossaire métier complet

**Documents qui ne te sont pas transmis** (gérés en interne) : Cahier des Charges (CDC), Product Requirements Document (PRD) détaillé, comptes rendus de décisions internes, listings d'open questions internes. Si tu découvres un manque d'information, signale-le en sync hebdo — on complétera.

---

## 2. Vue d'ensemble projet

### 2.1 Pitch

AXIA ISP Management Suite **centralise dans Odoo 18 LTS** la gestion commerciale et la décision métier de 4 marques télécom — **XIWO, WEELAX, COQLA, GLOBALGRID** — en synchronisant bidirectionnellement avec **4 instances Splynx** isolées (l'OSS/BSS réseau qui exécute PPPoE, RADIUS et coupures techniques).

> **Principe directeur** : *Odoo décide, Splynx exécute.* Toute décision métier (création client conditionnelle, suspension impayé, réactivation, résiliation) prend racine dans Odoo. Splynx remonte uniquement les états techniques. En gros : Odoo est le cerveau, Splynx les bras.

### 2.2 Découpage en chantiers

Le projet est découpé en chantiers livrés **séquentiellement, un à la fois**. Chaque chantier doit être accepté avant que le suivant ne démarre.

| Chantier | Objet | Critère acceptance global |
|---|---|---|
| 0 — Fondations (Sprint 0) | Bootstrap, audit, RBAC, POC Splynx, CI | Stack saine + audit + RBAC + pen test multi-tenant 100 % |
| 1 — Sync Odoo ↔ Splynx | Connecteur bidirectionnel complet | Création client < 5 min p95, sync bidirectionnelle, webhook |
| 2 — PPP & actions techniques | Gestion PPP + actions suspend/réactiver | Génération + rotation + révélation PPP gated, 0 rupture |
| 3 — Workflow impayés & billing | Automatisation impayés multi-pays | Scénarios Guadeloupe + Mali + multi-société E2E |
| 4 — Production-ready ops | Observabilité, archivage, RGPD | Dashboards + alerting + WORM + runbooks complets |

Chaque chantier dispose de son propre brief détaillé (scope, stories, acceptance, tests). **Tu reçois le brief du chantier suivant après acceptation du précédent.**

### 2.3 Discipline de livraison

- **1 chantier à la fois.** Le chantier N doit être signé-off avant le démarrage du chantier N+1.
- **Périmètre figé par chantier** une fois démarré. Toute évolution nécessite un ADR-amendment (§7).
- **Sign-off explicite** sur chaque chantier (§8.5).

Garde en tête que cette discipline nous évite le syndrome du scope qui gonfle — quand un chantier est validé, on passe au suivant sans revenir en arrière (sauf amendement).

### 2.4 Hors scope v1 (post-projet)

- IPTV, VoIP, services mobiles.
- Portail abonné final.
- Intégrations directes passerelles de paiement (GoCardless, Stripe…).
- Intégration directe fournisseur SMS (le système produit les triggers, l'envoi est porté par un module dédié).
- UI/UX pixel-perfect — les vues Odoo standard sont la cible.

---

## 3. Stack technique imposée

> 🔒 **Architecture figée.** Les choix résultent de 10 ADR validés (cf. `architecture.md` §Core Architectural Decisions). Toute déviation nécessite un ADR-amendment (§7). Ne t'inquiète pas, ça ne veut pas dire que rien ne peut bouger — juste qu'il faut suivre la procédure si besoin (§7).

### 3.1 Tableau stack

| Couche | Techno | Version | Pourquoi figé |
|---|---|---|---|
| ERP | **Odoo Community** | 18 LTS | Pas d'Enterprise (licence), pas d'Odoo.sh (on-premise imposé) |
| Base de données | **PostgreSQL** | 15 | Compat. Odoo 18 + partitioning natif via `pg_partman` |
| Cache & queue persistence | **Redis** | 7+ | Backend `queue_job` |
| Module connecteur | **OCA `connector`** | branche 18.0 | Pattern OCA standard, pas de SDK Splynx (licence floue) |
| Queue asynchrone | **OCA `queue_job`** | branche 18.0 | Channels multi-tenant, retry, throttling |
| Secrets at-rest | **OCA `keychain`** | branche 18.0 | Chiffrement Fernet, clé maître hors-base |
| Client HTTP sortant | **`httpx`** | 0.27+ | Wrapper minimal custom (~200 LOC, livré en Chantier 1) |
| Calendaires | **`python-holidays`** | 0.50+ | FR métropole + DOM-TOM + Mali + Sénégal |
| Tests | **pytest-odoo** | 0.8+ | Isolation par module |
| Lint | **black + isort + pylint-odoo** | dernières stables | + plugin pylint custom (couche 2 §4) |
| Conteneurs | **Docker + Compose** | 24+ | On-premise |
| Reverse proxy | **Nginx** | 1.25+ | TLS termination + IP allow-list |
| Observabilité (Chantier 4) | **Prometheus + Loki + Sentry** | latest | Mode pionnier — instrumentation J1 |
| Stockage WORM (Chantier 4) | **S3 + Object Lock** | — | Audit 5 ans glissants immuable |

### 3.2 Versions à pinner exactement

Les versions exactes des modules OCA et libs Python sont à figer au Sprint 0 (Chantier 0). **Aucune dépendance non pinnée** ne doit subsister à l'issue du Chantier 0. Petite astuce : un `pip freeze` ou `constraints.txt` fera très bien l'affaire.

### 3.3 Composants livrés en fin de Chantier 0 (vue globale)

```mermaid
flowchart TB
    subgraph host["Hôte Docker (on-premise)"]
        subgraph odoo["Odoo 18 LTS"]
            A0[Module Audit<br/>foundation : mixin + correlation + logger + storage partitionné]
            R0[Module RBAC<br/>7 rôles + record rules + décorateur gating]
            OCA[OCA connector + queue_job + keychain<br/>vendorisés]
        end
        PG[(PostgreSQL 15<br/>+ pg_partman)]
        RD[(Redis 7)]
    end
    
    subgraph ext["Externe (câblé Chantier 1+)"]
        SPLYNX[4 tenants Splynx]
    end
    
    A0 -. valid. POC .-> SPLYNX
    odoo --> PG
    OCA --> RD
```

Les 5 modules métier (sync, ppp, billing, admin params, observability) arrivent en Chantier 1 à 4. **Le scaffold canonique est documenté dans `architecture.md` §Project Structure.**

---

## 4. Onboarding développeur

Bon, on passe à la pratique ! Voici comment mettre les mains dans le cambouis.

### 4.1 Pré-requis machine

| Outil | Version min. |
|---|---|
| Docker | 24+ |
| Docker Compose | v2 (plugin) |
| Git | 2.40+ |
| Python (lint local) | 3.11+ |
| mitmproxy (Chantier 0 POC) | 10+ |

### 4.2 Premier démarrage

```bash
# 1. Cloner le repo
git clone <repo-url>
cd <repo-name>

# 2. Configurer les secrets (clé maître keychain obligatoire)
cp .env.example .env
# Renseigner au minimum la clé maître keychain et le mot de passe admin Odoo

# 3. Boot
docker compose up -d

# 4. Vérifier la santé
docker compose ps     # tous services healthy
curl http://localhost:8069/web/database/selector
```

L'instance doit être joignable sur `http://localhost:8069` en **< 5 min** sur poste neuf (re-boot), **< 15 min** cold-start (première fois, build image). Si ça prend plus longtemps, c'est probablement un souci de build — n'hésite pas à le remonter en sync.

### 4.3 Variables d'environnement clés

| Variable | Rôle | Obligatoire |
|---|---|---|
| Clé maître keychain (clé Fernet ≥ 32 chars) | Chiffrement secrets at-rest | **OUI** |
| Mot de passe master Odoo | Admin Odoo | **OUI** |
| Mot de passe PostgreSQL | DB | OUI |
| Mot de passe Redis | Cache | recommandé |
| Tokens API Splynx (1 par tenant) | POC Chantier 0 uniquement | Chantier 0 |
| DSN Sentry | Observabilité | Chantier 4 |
| Credentials S3 + bucket | Archivage WORM | Chantier 4 |

Les noms exacts sont au dev — listés dans `.env.example` que tu produiras en Story 0.1. **Aucun secret production ne doit être committé.** On ne le répétera jamais assez.

### 4.4 Workflow git & PR

| Élément | Convention |
|---|---|
| Branche principale | `main` (protégée) |
| Branche feature | par story, format au choix (suggestion : `feat/E{chantier}-S{story}-<slug>`) |
| Commits | Conventional Commits (`feat:`, `fix:`, `chore:`, `docs:`, `test:`) |
| PR | 1 PR par story, lien story en description, **DoD checklist obligatoire** (template §5.2) |
| Reviewers | ≥ 1 pair côté dev externe + sign-off Kelvin sur acceptance chantier |
| CI | Toutes les checks vertes obligatoires avant merge |

### 4.5 Tests locaux

```bash
# Suite tests d'un module
docker compose run --rm odoo pytest /mnt/extra-addons/<module>/tests/

# Lint avant commit (équivalent CI)
pre-commit run --all-files
```

---

## 5. Conventions transverses & naming load-bearing

> 📐 **Important — 3 couches de prescriptivité.** Petite aparté avant d'entrer dans le détail : ce projet utilise une discipline de spécification stratifiée. L'idée, c'est d'être strict là où ça compte vraiment, et de te laisser libre sur tout le reste.
>
> - **Couche 1 — Contrat (non négociable)** : ACs Given/When/Then, NFRs chiffrés, DoD, propriétés sécurité.
> - **Couche 2 — Naming load-bearing (justifié)** : un petit ensemble de conventions de nommage qui ne sont **pas** des préférences de style mais des **contrats trans-modules** ou des **contrôles de sécurité actifs**. Listés ci-dessous avec leur "pourquoi".
> - **Couche 3 — Implémentation (libre)** : tout le reste. Noms de méthodes, variables, fichiers internes, schémas SQL précis, idiomes Python — au dev (et son LLM).
>
> Si quelque chose ne figure ni dans la couche 1 ni dans la couche 2, **tu es libre**. Sérieusement. On te fait confiance sur le code.

### 5.1 Naming load-bearing — Couche 2

| Convention | Spécification | Pourquoi non négociable |
|---|---|---|
| **Format des identifiants d'événement d'audit** | `<domain>.<action>[.<modifier>]` lowercase, ≤ 64 caractères. Domaines minimums : `customer`, `service`, `ppp`, `suspension`, `reactivation`, `role`, `webhook`, `mapper`, `admin_param`, `field_overwrite`. | Si chaque dev choisit son format, l'audit 5 ans devient ingrep-able, le scan static CI ne marche plus, l'export RGPD perd sa cohérence. C'est un **contrat trans-modules**. |
| **Noms des 7 modules Odoo** | `axia_audit`, `axia_rbac`, `axia_splynx_connector`, `axia_ppp`, `axia_billing_workflow`, `axia_admin_params`, `axia_observability` | Référencés depuis 5 autres modules dans `__manifest__.py` deps + dans `architecture.md` figée + dans le graphe de dépendances. Renommer = casser la cohérence amont/aval. |
| **Nom du décorateur RBAC gating** | `@axia.requires_role(...)` (au minimum un identifiant scannable de manière déterministe par un plugin pylint custom) | Le scan static CI détecte les méthodes `action_*` sans gating. Si le nom du décorateur change sans MAJ du plugin, **bypass sécurité actif possible**. |
| **Nom de la table audit principale** | À utiliser dans le partitioning et les scripts archivage. Le nom doit rester stable car référencé dans le script d'init partitions, le job d'archivage Parquet, et les dashboards. | Convention archivage 5 ans s'appuie dessus. Suggestion : `axia_audit_event`. |

> **Note sur les channels queue_job** : leur nommage est **libre**, à ta discrétion, tant que l'invariant NFR-9 (sharding logique multi-tenant — chaque société a son channel isolé pour éviter qu'une marque congestionnée ne bloque les autres) et l'asymétrie de priorité (channels critiques pour payment/suspend traités en priorité haute) sont respectés. Documente ta convention de nommage dans `architecture.md` au démarrage du Chantier 1 (ADR-amendment léger : juste un PR doc).

**Tout autre nommage** (méthodes Python, variables, classes internes, helpers, fichiers de scripts, noms de colonnes JSONB internes au payload, etc.) est **libre**.

### 5.2 Conventions transverses (couche 1, observables)

| Domaine | Convention | Vérification |
|---|---|---|
| **Audit** | Toute mutation métier émet un événement d'audit. **Aucun INSERT direct** dans la table audit principale. | Tests + pen test |
| **Secrets** | Aucun secret en clair en base, en `ir.config_parameter`, en log, en variable d'env hors keychain, ni en repo. | Grep CI + audit logs |
| **Logs** | JSON structurés avec `correlation_id`, `level`, `module`, consommables par un agrégateur (Loki). | Grep CI |
| **Décorateur RBAC** | Toute méthode publique d'action sensible (suspendre, réactiver, révéler, exporter, supprimer) est gated par rôle. | Scan static + tests JSON-RPC bypass |
| **Tests par module** | Un dossier `tests/` par module, exécutable isolément. | CI parallèle |
| **Manifests** | Déclarent les deps OCA et inter-modules de manière explicite. | Boot Odoo |
| **Aucun `print()` ni `logging.getLogger` direct** | Utiliser le wrapper logger fourni par le module audit. | Grep CI |

### 5.3 Anti-patterns explicitement bannis (cf. architecture.md §Anti-Patterns)

- ❌ OCA `auditlog` sur le hot path (perf)
- ❌ Stockage de secret en clair où que ce soit
- ❌ SDK Splynx tiers (R8 licence)
- ❌ Suppression physique d'un enregistrement Splynx (toujours `terminated`)
- ❌ Booléen flag `create_to_radius` sur res.partner pour piloter la création (refonté en pattern binding + événement métier)
- ❌ Long-running batch monolithique (jobs granulaires)

---

## 6. Definition of Done

> ⚠️ **8 critères, applicables à chaque story**. On ne merge **aucune PR** si un seul critère n'est pas vert. C'est binaire — et c'est ce qui nous garantit de ne pas accumuler de dette. À reporter dans la description de chaque PR en checklist.

### 6.1 Les 8 critères

- [ ] **1. Code conforme** aux conventions OCA et aux conventions §4 (naming load-bearing respecté, anti-patterns évités).
- [ ] **2. Tests unitaires** : couverture > 80 % sur le code nouveau ou modifié. Méthodes pures, validations, services.
- [ ] **3. Tests intégration** avec DB (`pytest-odoo`) qui valident les Acceptance Criteria Given/When/Then de la story.
- [ ] **4. Événement d'audit déclaré** : si la story émet de nouveaux types d'événements d'audit, ils sont déclarés dans l'inventaire centralisé. Sinon « N/A ».
- [ ] **5. Pre-commit OK** : black, isort, pylint-odoo, plugin lint custom RBAC tous verts.
- [ ] **6. Doc utilisateur** : si comportement opérateur change, mise à jour `docs/`. Sinon « N/A ».
- [ ] **7. CI verte** : tests parallèles + grep secrets + pen test cross-company (nightly) inchangés.
- [ ] **8. Acceptance Criteria validés** : tous les Given/When/Then de la story passent en revue par un pair.

### 6.2 Template description de PR

```markdown
## Story
E<chantier>.S<story> — <titre>

## Couverture
- FR-XX, NFR-XX (cf. brief chantier)

## Changements (3-5 bullets)

## DoD checklist
- [ ] 1. Code conforme conventions
- [ ] 2. Tests unitaires (coverage > 80 % nouveau code)
- [ ] 3. Tests intégration valident les ACs
- [ ] 4. Inventaire événements audit mis à jour (ou N/A)
- [ ] 5. Pre-commit OK
- [ ] 6. Doc utilisateur (ou N/A)
- [ ] 7. CI verte
- [ ] 8. ACs validés par reviewer pair

## Démo
<comment / commande / screenshot>

## Notes pour le reviewer
```

---

## 7. Règles de déviation architecture (ADR-amendment)

> 🔒 L'architecture (10 ADR + 7 modules + stack) est **figée**. Cela dit, on n'est pas dans le dogme : tu **peux** proposer une déviation si tu découvres un blocage technique réel, mais tu **dois** suivre la procédure ci-dessous. **Jamais d'amendement silencieux dans le code.** Si tu contournes sans le dire, on va le voir — et ça fait perdre du temps à tout le monde.

### 7.1 Quand proposer un ADR-amendment

| Situation | Action |
|---|---|
| Un module OCA imposé ne supporte plus Odoo 18 (bug bloquant) | ADR-amendment obligatoire |
| Une API Splynx découverte en POC contredit nos hypothèses | ADR-amendment obligatoire |
| Une perf NFR ne tient pas en charge sur la techno choisie | ADR-amendment obligatoire |
| Tu préfères `pytest` à `pytest-odoo` | ❌ Pas un amendement, non négociable |
| Tu trouves plus joli d'utiliser une lib alternative | ❌ Pas un amendement |
| Tu veux ajouter une dépendance Python | RFC léger en sync hebdo, pas d'amendement complet |

> **Règle d'or** : un amendement existe pour les **blocages techniques avérés**, pas pour les préférences de style. En gros : si c'est juste une question de goût, on garde la stack telle quelle.

### 7.2 Format

Crée `docs/adr/ADR-XXX-amendment-<slug>.md` :

```markdown
# ADR-XXX-amendment-<slug>

**Date** : YYYY-MM-DD
**Auteur** : <nom>
**ADR modifié** : ADR-00X
**Statut** : Proposed | Accepted | Rejected

## Contexte
<blocage rencontré, faits + données mesurées>

## Décision proposée
<modification précise demandée>

## Conséquences
<impact sur autres ADRs, stories restantes, calendrier>

## Alternatives évaluées
<ce que tu as essayé, pourquoi ça ne marche pas>

## Sign-off requis
- [ ] Kelvin (décideur produit + architecte interne)
- [ ] Reviewer pair côté dev externe
```

### 7.3 Workflow d'approbation

1. PR draft avec l'ADR-amendment + preuve du blocage (logs, bench, reproduce steps).
2. Mention Kelvin en reviewer.
3. Discussion en sync hebdo si non-trivial.
4. Décision finale Kelvin sous 72 h ouvrées.
5. Si accepté : merge l'ADR-amendment **avant** modification de code applicatif.
6. Si rejeté : implémenter selon ADR initial. Le rejet est tracé.

### 7.4 Incohérence détectée dans un brief

Si tu trouves une contradiction entre ce brief commun, le brief de chantier, et `architecture.md` : **stoppe la story concernée et signale-le immédiatement** en sync hebdo ou par message direct. Ne devine pas l'intention. Mieux vaut poser la question que partir dans la mauvaise direction — on est là pour ça.

---

## 8. Ressources & contacts

> 🟡 Cette section contient des `[TBD - Kelvin]` à compléter **avant remise** au prestataire. Certains champs seront vides quand tu liras ce document pour la première fois, c'est normal — ils seront remplis avant le premier commit.

### 8.1 Repo & artefacts

| Ressource | URL / chemin |
|---|---|
| Repo principal | `[TBD - Kelvin]` |
| Registre Docker (images release) | `[TBD - Kelvin]` |
| Bucket S3 archivage WORM (Chantier 4) | `[TBD - Kelvin]` |
| Dashboard projet (Linear / Jira / GitHub Projects) | `[TBD - Kelvin]` |

### 8.2 Accès Splynx (Chantier 0 — POC payloads)

| Tenant | URL | Compte API dédié | Canal transmission token sécurisé |
|---|---|---|---|
| XIWO | `[TBD]` | `[TBD]` | Bitwarden / 1Password share |
| WEELAX | `[TBD]` | `[TBD]` | idem |
| COQLA | `[TBD]` | `[TBD]` | idem |
| GLOBALGRID | `[TBD]` | `[TBD]` | idem |

**Les tokens ne sont jamais transmis par email/Slack/Discord en clair.** Canal sécurisé obligatoire.

### 8.3 Contacts

| Rôle | Personne | Canal | Pour quoi |
|---|---|---|---|
| Décideur produit + architecte interne | Kelvin | `[TBD - email]` + sync hebdo | Décisions ADR-amendment, validation acceptance, levée blocages |
| Tech lead côté dev externe | `[TBD]` | `[TBD]` | Coordination interne équipe |
| Support Splynx (bug API tenant) | `[TBD]` | `[TBD]` | Bugs spécifiques tenants |

### 8.4 Cadence

| Événement | Fréquence | Format |
|---|---|---|
| Sync planning | Lundi 10 h | 30 min — backlog week + blocages |
| Daily async | Tous les jours | Message court canal projet — état stories |
| Sync technique (au besoin) | Mercredi 14 h | 1 h — débrief blocages + ADR-amendments |
| Démo fin de chantier | Fin chantier N | 1 h — démo staging + sign-off acceptance |

### 8.5 Sign-off acceptance par chantier

Un chantier est **officiellement accepté** uniquement après :

1. ✅ Tous les critères d'acceptance globale du chantier validés sur env staging.
2. ✅ Démo réalisée devant Kelvin.
3. ✅ Sign-off écrit Kelvin sur la PR finale du chantier (commentaire `LGTM — Chantier N accepté`).
4. ✅ Aucune issue critical ni major ouverte sur le scope du chantier.

À partir de ce sign-off, le périmètre du chantier devient figé (modifications via ADR-amendment uniquement). Le chantier N+1 démarre et son brief te sera remis. Le cycle recommence !

---

## Annexe — Glossaire express

| Terme | Définition |
|---|---|
| AXIA | Nom du projet ISP Management Suite |
| ISP | Internet Service Provider (fournisseur d'accès Internet) |
| Splynx | OSS/BSS télécom de référence ; 4 tenants opérés (1 par marque) |
| PPP / PPPoE | Point-to-Point Protocol (over Ethernet) — protocole d'authentification abonné réseau |
| Backend | Une instance Splynx = un backend AXIA (1 par marque) |
| Binding | Pattern OCA : modèle Odoo qui matérialise le lien (record local, système distant) |
| `correlation_id` | UUID propagé end-to-end pour reconstituer un cycle de décision |
| Outbox / Inbox | Pattern transactionnel pour idempotence des intégrations (Chantier 1) |
| Record rule | Mécanisme Odoo de filtrage automatique des records par utilisateur/société |
| WORM | Write Once Read Many — stockage immuable (S3 Object Lock) pour archivage 5 ans |
| ADR | Architecture Decision Record |
| OCA | Odoo Community Association — communauté open source Odoo |
| Tenant | Une instance applicative cloisonnée (s'applique aux instances Splynx) |
