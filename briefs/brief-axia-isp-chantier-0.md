---
title: Brief Chantier 0 — Sprint de Fondation
project: AXIA ISP Management Suite
chantier: 0
audience: Équipe développement externe
status: Ready for handoff
date: 2026-06-11
version: 3.0
prerequisite: brief-axia-isp-commun.md
---

# Brief Chantier 0 — Sprint de Fondation

> 📖 **Document compagnon** : ce brief s'utilise **avec** [`brief-axia-isp-commun.md`](./brief-axia-isp-commun.md) qui contient la stack figée, les conventions transverses, la Definition of Done, le workflow d'ADR-amendment et les ressources. **Lis-le d'abord.**

## Sommaire

1. [Objet du chantier](#1-objet-du-chantier)
2. [Scope](#2-scope)
3. [Acceptance globale](#3-acceptance-globale)
4. [Les 10 stories](#4-les-10-stories)
5. [Tests d'acceptation Chantier 0](#5-tests-dacceptation-chantier-0)

---

## 1. Objet du chantier

Un sprint de fondation, 100 % technique. On va poser les briques horizontales — module **audit** (storage partitionné + mixin + propagation `correlation_id` + logger structuré), module **RBAC** (7 rôles + record rules cross-company), intégration du secret store, POC payloads Splynx sur les 4 tenants, pipeline CI — sur lesquelles les chantiers métier (1 à 4) vont s'appuyer. Aucune feature métier n'est livrée à ce stade.

---

## 2. Scope

- Bootstrap du repo (Docker Compose, dépendances pinnées, scaffold modules, CI)
- Module **audit** : stockage partitionné + mixin + propagation `correlation_id` + logger structuré
- Module **RBAC** : 7 rôles + record rules cross-company strictes + décorateur de gating + wizard délégation partenaires
- POC payloads Splynx sur les 4 tenants (golden files de référence ; clôt l'open question OQ-7)
- Intégration du secret store (`keychain`) opérationnelle
- Test de pénétration cross-company automatisé
- Pipeline CI complète (pre-commit + tests parallèles par module + build image + nightly pen test)

---

## 3. Acceptance globale

Le chantier est accepté **uniquement si** les 6 critères suivants sont tous au vert sur l'env de staging :

1. ✅ `docker compose up` sur poste neuf produit une stack saine (Odoo + DB + queue backend + observabilité stub) en < 15 min sans intervention manuelle.
2. ✅ Module audit installable et opérationnel : storage partitionné, mixin testable, `correlation_id` propagé end-to-end, inventaire d'événements validé au boot.
3. ✅ Module RBAC installable : 7 rôles créés, record rules cross-company actives, décorateur de gating rejette server-side, scan static détecte les méthodes sensibles sans gating.
4. ✅ POC payloads Splynx : ≥ 40 golden files JSON capturés sur les 4 tenants (5 opérations × 4 backends × 2 minimum), matrice `field-mappings.md` à jour, ADR-001 (contrat Mapper & POC payloads Splynx) signé.
5. ✅ **Pen test cross-company : 100 % couverture, 0 fuite détectée** (UI + JSON-RPC direct).
6. ✅ CI verte sur la PR finale : pre-commit + tests par module en parallèle + build image + pen test nightly.

---

## 4. Les 10 stories

> Format light : `As a / I want / So that` + ACs comportementaux Given/When/Then + couverture FR/NFR/ADR + effort. L'implémentation (noms de classes, méthodes, fichiers internes) est à ta discrétion — seul le comportement observable est contractuel.

### Story E0.S1 — Bootstrap repo & stack reproductible

**As a** lead développeur,
**I want** un repo bootstrapé avec une stack reproductible et documentée,
**So that** un dev nouveau-arrivant obtient une instance saine en moins de 15 minutes.

**Couvre** : Conventions §4 et §5 du brief commun.
**Effort** : 3 jours.

**Acceptance Criteria** :
- **Given** un poste de dev avec Docker installé,
- **When** un dev clone le repo et exécute la procédure de boot documentée,
- **Then** Odoo, la base de données, le backend de queue et la stub d'observabilité sont sains,
- **And** les dépendances tierces sont pinnées sur des versions identifiables (clôt OQ-6),
- **And** les variables d'environnement requises sont documentées avec un template,
- **And** la procédure et les pré-requis sont décrits dans un README.

---

### Story E0.S2 — Storage audit partitionné

**As a** dev backend,
**I want** un storage d'événements d'audit dimensionné pour la rétention 5 ans à 100 k abonnés,
**So that** la consultation chaude reste performante et l'archivage froid est faisable.

**Référence architecture** : ADR-005 (schéma audit & partitionnement).
**Effort** : 4 jours.

**Acceptance Criteria** :
- **Given** l'installation fraîche du module audit,
- **When** le module est installé,
- **Then** le storage est partitionné mensuellement avec partitions pré-créées sur 13 mois glissants,
- **And** les indexes nécessaires aux usages décrits dans ADR-005 (schéma audit & partitionnement) sont en place (consultation par client, par event_type, par période),
- **And** les droits d'accès au modèle sont restreints : `create` et `read` seulement, `write`/`unlink` réservés à un groupe administrateur dédié,
- **And** une procédure de maintenance des partitions (cron) est documentée.

---

### Story E0.S3 — Mixin audit + propagation `correlation_id` + logger structuré

**As a** dev backend de tout module AXIA,
**I want** un point d'entrée unique pour émettre un événement d'audit, propager un `correlation_id` end-to-end, et émettre des logs structurés sans secrets,
**So that** tout chemin de code mutatoire est tracé de manière homogène.

**Référence architecture** : ADR-005 (schéma audit & partitionnement), conventions §5 brief commun.
**Effort** : 4 jours.

**Acceptance Criteria** :
- **Given** un module consommateur intégrant l'API audit fournie,
- **When** il émet un événement d'audit avec un payload,
- **Then** un enregistrement est créé avec le `correlation_id` du contexte actif,
- **And** le payload respecte le schéma standard (acteur, cible, delta) + extensions déclarées,
- **And** un événement avec un type non déclaré dans l'inventaire centralisé lève une exception immédiate,
- **Given** un point d'entrée (controller, cron, action UI) wrappé dans un contexte de corrélation,
- **When** ce code émet des logs structurés,
- **Then** chaque log contient le `correlation_id` propagé,
- **And** la propagation traverse l'enqueue/dequeue de jobs asynchrones,
- **And** aucun secret (recherche : `pass_ppp`, `token`, `password`, `secret`) n'apparaît dans la sortie logs (vérifié par grep CI).

---

### Story E0.S4 — Inventaire centralisé des événements d'audit + validation au boot

**As a** dev backend,
**I want** un inventaire centralisé de tous les types d'événements d'audit du projet, validé au démarrage,
**So that** aucun chemin de code ne peut émettre un événement non déclaré (mode pionnier, anti-drift).

**Couvre** : Convention §5.1 du brief commun (format event_type).
**Effort** : 2 jours.

**Acceptance Criteria** :
- **Given** le module audit installé,
- **When** Odoo démarre,
- **Then** l'inventaire d'événements est chargé : pour chaque entrée, type + description + schéma de payload + module propriétaire,
- **And** l'inventaire couvre au minimum les domaines décrits dans la convention §4.1,
- **When** un consommateur tente d'émettre un événement non déclaré,
- **Then** une exception est levée immédiatement,
- **And** un scan static (CI) vérifie que tous les types d'événements cités dans le code sont déclarés dans l'inventaire.

---

### Story E0.S5 — Module RBAC : 7 rôles + record rules cross-company strictes

**As a** admin fonctionnel,
**I want** 7 rôles configurés sous forme de groupes Odoo avec record rules cross-company strictes,
**So that** chaque utilisateur ne voit que les données de sa propre marque (XIWO/WEELAX/COQLA/GLOBALGRID) avec la visibilité métier appropriée.

**Référence architecture** : ADR-003 (multi-company natif).
**Effort** : 5 jours.

Les 7 rôles (hiérarchie d'implications cf. PRD §3.7) : Commercial, Administratif, Contentieux, Contentieux & Administratif, Responsable de marque, Direction Groupe, Administrateur Odoo.

**Acceptance Criteria** :
- **Given** 4 sociétés créées (XIWO, WEELAX, COQLA, GLOBALGRID),
- **When** un utilisateur Commercial rattaché à XIWO interroge l'ORM (UI ou JSON-RPC direct),
- **Then** il ne voit **aucun** record d'une autre société, sur aucun modèle métier,
- **And** la matrice de visibilité par champ (matrice de visibilité dans la SPEC) est respectée selon le rôle (ex. notes commerciales OK pour Commercial mais notes administratives masquées),
- **And** la hiérarchie d'implications fonctionne (Contentieux & Administratif voit l'union de Contentieux + Administratif ; Direction Groupe a tout sauf Administrateur Odoo orthogonal),
- **And** un wizard permet à l'admin fonctionnel de créer un profil dérivé cloisonné (centre d'appel, sous-traitant) sans intervention dev.

---

### Story E0.S6 — Décorateur de gating par rôle + scan static custom

**As a** dev backend,
**I want** un mécanisme de gating server-side par rôle, scannable statiquement par un outil de lint,
**So that** le bypass via JSON-RPC est bloqué ET le scan détecte tout oubli de gating sur une méthode sensible.

**Couvre** : **Conventions** : §5.1 brief commun.
**Effort** : 3 jours.

**Acceptance Criteria** :
- **Given** une méthode publique d'action sensible gated par rôle,
- **When** un utilisateur hors périmètre l'appelle (UI ou JSON-RPC direct),
- **Then** l'appel est rejeté avec une `AccessError`,
- **And** un événement d'audit `role.access.denied` est émis (acteur, méthode, rôle requis),
- **Given** un utilisateur dans le périmètre,
- **When** il appelle la méthode,
- **Then** elle s'exécute,
- **And** un scan static (CI) détecte toute méthode publique d'action sensible **sans** gating (warning bloquant la PR).

---

### Story E0.S7 — Test de pénétration cross-company automatisé

**As a** responsable sécurité,
**I want** un test de pénétration cross-company automatisé,
**So that** je peux **prouver à 100 %** qu'aucune fuite multi-tenant n'existe (R13, ADR-003 — multi-company natif).

**Référence architecture** : ADR-003 (multi-company natif).
**Effort** : 3 jours.

**Acceptance Criteria** :
- **Given** une instance peuplée avec 4 sociétés + démo data réaliste (≥ 10 partners + 5 événements d'audit par société),
- **When** le test tourne pour chaque rôle non-cross-company,
- **Then** l'inventaire des records visibles pour ce rôle (UI ET JSON-RPC `search_read` sans `domain`) **n'inclut aucun record d'une autre société**,
- **And** tentative de `read` sur un record d'autre société par ID explicite lève `AccessError`,
- **And** tentative de `write` ou `unlink` sur un record d'autre société lève `AccessError`,
- **And** le test génère un rapport machine-readable consommable par la CI,
- **And** le test échoue (exit non-zero) si au moins une fuite est détectée,
- **And** le test est intégré dans le pipeline CI nightly.

---

### Story E0.S8 — POC payloads Splynx — golden files sur 4 tenants

**As a** dev backend connecteur,
**I want** capturer les payloads JSON réels échangés avec les 4 tenants Splynx pour les opérations clés,
**So that** le contrat Mapper du Chantier 1 repose sur des données réelles et non sur des suppositions (clôt OQ-7).

**Référence architecture** : ADR-001 (contrat Mapper & POC payloads Splynx).
**Effort** : 5 jours (incluant accès tenants + capture + documentation).

**Acceptance Criteria** :
- **Given** les 4 tenants Splynx accessibles avec compte API dédié (cf. §8.2 brief commun),
- **When** le POC exécute les 5 opérations sur chaque tenant via un intercepteur (mitmproxy ou équivalent) : `POST customer`, `POST internet-service`, `PUT customer`, `PUT internet-service`, lecture monitoring,
- **Then** ≥ 2 golden files JSON par opération par tenant sont stockés (≥ 40 fichiers au total),
- **And** chaque payload est documenté : champs requis vs optionnels, types, formats `additional_attributes`, nomenclature `latitude/longitude/gps`,
- **And** la matrice `field-mappings.md` est mise à jour avec la nomenclature réelle observée,
- **And** un ADR-001 (contrat Mapper & POC payloads Splynx) signé synthétise les divergences entre tenants (s'il y en a) et tranche le contrat Mapper v1,
- **And** la procédure de capture est documentée pour les futurs ajouts de tenants.

> ⚠️ **Petit point important** : il te faut l'accès aux 4 tenants Splynx + un compte API dédié. Voir §8.2 brief commun.

---

### Story E0.S9 — Intégration secret store opérationnel

**As a** dev backend,
**I want** un secret store chiffré at-rest, avec clé maître hors-base, opérationnel pour tous les secrets du projet,
**So that** dès le bootstrap (avant la première feature qui consommera un token).

**Référence architecture** : ADR-007 (auth API), ADR-004 (PPP).
**Effort** : 3 jours.

**Acceptance Criteria** :
- **Given** une instance avec la clé maître du secret store positionnée en variable d'environnement,
- **When** un admin enregistre un secret via l'UI du secret store,
- **Then** la valeur est stockée chiffrée en base (vérifiable : `SELECT` brut sur la table ne révèle pas la valeur),
- **And** un service backend peut lire le secret par son identifiant sans matérialiser la valeur dans les logs,
- **And** la rotation manuelle du secret est possible via l'UI,
- **And** un test E2E vérifie qu'un grep sur les logs après cycle complet d'accès ne retourne aucune valeur de secret en clair,
- **And** des comptes/secrets pré-déclarés couvrent les besoins connus : 4 tokens API Splynx + 4 secrets HMAC webhook + credentials S3 + DSN Sentry + clé chiffrement PPP.

---

### Story E0.S10 — Pipeline CI complète

**As a** lead développeur,
**I want** une pipeline CI qui exécute pre-commit, tests par module en parallèle, et release Docker sur tag,
**So that** chaque PR est validée automatiquement et un tag de release produit une image déployable.

**Couvre** : Discipline tests + CI (§6 brief commun).
**Effort** : 3 jours.

**Acceptance Criteria** :
- **Given** une PR poussée,
- **When** la CI tourne,
- **Then** les étapes minimum sont : pre-commit (black, isort, pylint-odoo, plugin RBAC) + tests par module en parallèle + test de pénétration cross-company (Story E0.S7) + grep secrets dans les logs,
- **And** la PR est bloquée si une étape échoue,
- **Given** un tag de release poussé,
- **When** la pipeline release tourne,
- **Then** une image Docker (nom et tag au choix selon votre convention) est buildée et poussée sur un registre privé configuré,
- **And** un premier tag de release démontre le release path à la fin du Chantier 0.

---

## 5. Tests d'acceptation Chantier 0

🟡 **Avant de démarrer** : on va te fournir un set de tests d'acceptation **red-phase exécutables** (ATDD) qui couvre les 6 critères d'acceptance globale (§3) et chaque AC de story. **Ton taf : les faire passer au vert sans toucher à leur logique.** C'est le **contrat dur** de livraison.

| Catégorie | Story source | Type de vérification attendue |
|---|---|---|
| Stack healthcheck | S1 | Smoke E2E : containers healthy + Odoo répond |
| Audit storage | S2 | Storage partitionné, indexes, droits d'accès |
| Mixin & propagation | S3 | Émission, validation payload, propagation correlation, no-leak secrets |
| Inventaire événements | S4 | Inventaire chargé, validation, scan static |
| RBAC strict | S5 | Visibilité par rôle, hiérarchie implications, wizard délégation |
| Gating décorateur | S6 | Deny/allow + audit + scan static |
| Pen test cross-company | S7 | 100 % couverture, 0 fuite |
| POC Splynx | S8 | ≥ 40 golden files, ADR-001 (contrat Mapper & POC payloads Splynx) signé |
| Secret store | S9 | Chiffrement at-rest, no-leak logs, rotation |
| Pipeline CI | S10 | Toutes étapes vertes, release tag → image |

---

**Fin du Brief Chantier 0.**

> Si une question te reste en suspens, pose-la en sync hebdo **avant** d'écrire la première ligne de code de la story concernée. Bon chantier. 🏗️
