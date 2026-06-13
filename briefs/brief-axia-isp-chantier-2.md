---
title: Brief Chantier 2 — PPP & actions techniques
project: AXIA ISP Management Suite
chantier: 2
audience: Équipe développement externe
status: Ready for handoff
date: 2026-06-11
version: 3.0
prerequisite: brief-axia-isp-commun.md
chantier-prerequisite: Chantier 1 signé-off
---

# Brief Chantier 2 — PPP & actions techniques

> 📖 **Document compagnon** : ce brief s'utilise **avec** [`brief-axia-isp-commun.md`](./brief-axia-isp-commun.md). **Lis-le d'abord.**
>
> 🔒 **Prérequis** : Chantier 1 (Synchronisation Odoo ↔ Splynx) accepté et signé-off. Les bindings customer & internet-service, le client HTTP, le Mapper v1 et les webhooks doivent être opérationnels.

## Sommaire

1. [Objet du chantier](#1-objet-du-chantier)
2. [Scope](#2-scope)
3. [Acceptance globale](#3-acceptance-globale)
4. [Les 14 stories](#4-les-14-stories)
5. [Tests d'acceptation Chantier 2](#5-tests-dacceptation-chantier-2)

---

## 1. Objet du chantier

C'est parti pour le Chantier 2. On attaque le module **PPP** de A à Z : génération Odoo-owned, rotation gated, révélation gated, chiffrement at-rest, import legacy passif. Et on complète le connecteur avec les **actions techniques** — suspendre, réactiver, résilier — le tout gated par rôle.

Petit rappel : l'inversion d'ownership PPP, c'est tranché dans le CDC v2. **Odoo génère** les identifiants et mots de passe PPPoE, **Splynx les applique**. Point.

---

## 2. Scope

Concrètement, voilà ce qu'on met en place dans ce chantier :

- Modèle de stockage PPP par service Internet (login en clair + mot de passe chiffré at-rest)
- Générateur PPP (entropie cryptographique, format conforme ADR-004 — génération PPP Odoo-owned)
- Pre-check anti-collision Splynx avant tout POST (retry borné + escalade en arbitrage humain)
- Génération automatique du PPP à la création d'un service Internet + propagation en un round-trip vers Splynx
- Wizard rotation PPP gated par rôle (avec audit + purge ancien mot de passe)
- Wizard révélation `Pass ppp` gated par rôle (auto-masquage après 30 s + audit)
- Application de la matrice de visibilité par rôle sur les champs sensibles (matrice de visibilité dans la SPEC)
- Import legacy **passif** des PPP existants côté Splynx (ADR-002 — reprise PPP legacy — pas de rotation forcée)
- Snapshot baseline S3 WORM des PPP legacy (rollback de secours)
- Dashboard kanban Odoo « PPP legacy vs Odoo-generated »
- Actions techniques `suspend`, `reactivate`, `terminate` avec gating server-side par rôle
- Détection asymétrie blocage Splynx (action manuelle non passée par Odoo) + alerte
- Runbooks 01 (asymétrie blocage) et 04 (procédure import legacy)

---

## 3. Acceptance globale

Ce chantier est bon pour le sign-off quand :

1. ✅ La création d'un service Internet déclenche la génération du PPP côté Odoo et sa propagation à Splynx **en un seul round-trip** (pas de PUT ultérieur).
2. ✅ Rotation et révélation du `Pass ppp` sont gated par rôle (Contentieux & Administratif, Direction Groupe, Administrateur Odoo) et chaque opération émet un événement d'audit dédié (`ppp.rotated`, `ppp.revealed`).
3. ✅ **Tous** les `Pass ppp` sont chiffrés at-rest via le secret store ; jamais en clair en base, dans les logs, ni dans les backups (vérifié par grep CI + tests).
4. ✅ Import legacy passif : 100 % des services Splynx existants sont importés dans le storage PPP local, **zéro rupture PPPoE**, snapshot S3 WORM archivé.
5. ✅ Actions techniques `suspend` / `reactivate` / `terminate` exécutables depuis Odoo, gated par rôle conformément à la matrice matrice de gating par rôle dans la SPEC.
6. ✅ Le scénario PRD §15.4 (réactivation 24/7) passe et le test E2E PPP cycle de vie passe.

---

## 4. Les 14 stories

On entre dans le vif du sujet — les 14 stories qui composent ce chantier.

### Story E2.S1 — Storage PPP chiffré at-rest

**As a** dev backend PPP,
**I want** un modèle de stockage qui pour chaque service Internet associe un login PPP (en clair, c'est un identifiant) et une référence chiffrée vers le mot de passe (jamais en clair en base),
**So that** chaque service a son couple PPP, le mot de passe chiffré at-rest via le secret store.


**Acceptance Criteria** :
- **Given** un service Internet créé, **When** le code génère un PPP et stocke le mot de passe, **Then** le mot de passe est chiffré via le secret store (clé maître hors-base) et seule la référence chiffrée est en base,
- **And** une requête SQL brute sur la table ne révèle jamais la valeur en clair,
- **And** la récupération du mot de passe en clair n'est possible que si l'appelant a passé un gating de rôle (audit `ppp.revealed` émis avant retour à l'UI — cf. Story E2.S6).

---

### Story E2.S2 — Générateur PPP (login + mot de passe haute entropie)

**As a** dev backend PPP,
**I want** un service qui produit un couple `(login, mot_de_passe)` conforme à ADR-004 (génération PPP Odoo-owned) : login `<marque-slug>-<base32-8-chars>` (entropie ≥ 40 bits), mot de passe 24 caractères alphanumériques (`ascii_letters + digits`, entropie ≥ 80 bits),
**So that** l'inversion d'ownership PPP (CDC v2) est opérationnelle avec qualité cryptographique.

**Référence architecture** : ADR-004 (génération PPP Odoo-owned).

**Acceptance Criteria** :
- **Given** un backend (ex. Marque A), **When** le générateur est invoqué, **Then** le login correspond à la regex `^marque-a-[a-z2-7]{8}$`, le mot de passe à `^[A-Za-z0-9]{24}$`,
- **And** 10 000 générations consécutives produisent ≥ 99,9 % de logins distincts (test statistique),
- **And** l'entropie du mot de passe est ≥ 80 bits (test de Shannon),
- **And** le service est pluggable (la stratégie de génération est configurable sans redéploiement, si Splynx impose une contrainte future).

---

### Story E2.S3 — Pre-check anti-collision Splynx + retry borné

**As a** dev backend PPP,
**I want** qu'avant tout POST de création vers Splynx, un GET vérifie qu'aucun service existant n'utilise déjà le login candidat ; en cas de collision, régénération avec compteur borné (max 3 essais) ; au-delà, escalade pour arbitrage humain,
**So that** le risque de doublons de login est nul.


**Acceptance Criteria** :
- **Given** 3 collisions consécutives simulées, **Then** un événement d'audit `ppp.collision.unresolved` est émis (binding, motif, candidats testés),
- **And** le binding reste en état « en attente d'arbitrage » et un écran liste les cas,
- **Given** une collision simple suivie d'un succès, **Then** retry transparent, événement d'audit `ppp.collision.resolved` (niveau info).

---

### Story E2.S4 — Génération PPP à la création de service + propagation en un round-trip

**As a** dev backend PPP,
**I want** que la création d'un service Internet côté Odoo (déclencheur binding) génère le couple PPP **avant** le POST Splynx, de sorte que le service soit créé chez Splynx avec son PPP en un seul appel,
**So that** Odoo génère et Splynx applique sans round-trip supplémentaire (pas de PUT ultérieur pour positionner le PPP).


**Acceptance Criteria** :
- **Given** un service Internet nouveau côté Odoo, **When** le binding est créé, **Then** un binding PPP est créé dans la même transaction (`source='odoo'`),
- **And** le payload Mapper inclut le login et le mot de passe (le mot de passe en clair existe **uniquement** dans le payload sortant en mémoire, jamais loggé, redacté dans les événements d'audit),
- **And** un **seul** POST Splynx crée customer + service + PPP en un round-trip,
- **And** un événement d'audit `ppp.generated` est émis (binding, login en clair, mot de passe redacted).

---

### Story E2.S5 — Wizard rotation PPP gated par rôle

**As a** opérateur Contentieux & Administratif,
**I want** un bouton « Régénérer PPP » sur la fiche service qui ouvre un wizard de confirmation explicite (rupture PPPoE active à attendre),
**So that** la rotation est tracée.


**Acceptance Criteria** :
- **Given** un user Commercial, **Then** le bouton « Régénérer PPP » est invisible,
- **Given** un user Contentieux & Administratif, **When** il clique sur le bouton et confirme, **Then** un nouveau couple est généré, la propagation vers Splynx est asynchrone,
- **And** un événement d'audit `ppp.rotated` est émis (delta : ancien login → nouveau login, mots de passe redacted),
- **And** l'ancien mot de passe est purgé du secret store à T+0,
- **And** l'UX prévient l'opérateur : « La session PPPoE active sera renégociée. »

---

### Story E2.S6 — Wizard révélation du `Pass ppp` gated + audit

**As a** opérateur Contentieux & Administratif,
**I want** que le `Pass ppp` soit affiché masqué par défaut, et qu'un bouton « Révéler » ouvre un wizard qui demande confirmation puis affiche le clair pendant 30 secondes avant de re-masquer automatiquement,
**So that** chaque révélation est tracée **avant** affichage.


**Acceptance Criteria** :
- **Given** un user Commercial, **Then** le champ `Pass ppp` est masqué et le bouton « Révéler » est invisible,
- **Given** un user Contentieux & Administratif, **When** il clique « Révéler » et confirme, **Then** un événement d'audit `ppp.revealed` est émis **avant** le retour à l'UI (auteur, service, horodatage),
- **And** le clair est affiché pendant 30 s puis re-masqué automatiquement,
- **And** la valeur n'est **jamais** transmise via URL ni stockée côté frontend persistant.

---

### Story E2.S7 — Matrice de visibilité par rôle sur les champs sensibles

**As a** admin fonctionnel,
**I want** que la matrice de visibilité matrice de visibilité dans la SPEC soit appliquée par rôle sur les champs sensibles (notes commerciales, notes administratives, notes contentieux, champs techniques PPP / SIM / WIFI / L2TP / VOIP, procédures contentieuses, adresse complète),
**So that** un Commercial ne voit ni notes administratives ni `Pass ppp`, mais voit ses notes commerciales et les coordonnées de base.


**Acceptance Criteria** :
- **Given** un user Commercial, **Then** les champs `notes_admin`, `notes_contentieux`, `pass_ppp`, `sim_puk`, `wifi_password`, `l2tp_secret`, `voip_password`, `procedures_contentieux` sont **absents** de la vue formulaire (filtrés par groupe),
- **And** une tentative `search_read` via JSON-RPC sur ces champs est rejetée (`AccessError`),
- **Given** un user Contentieux & Administratif, **Then** tous ces champs sont visibles.

---

### Story E2.S8 — Import legacy passif des PPP existants

**As a** ops migration,
**I want** un job lancé une fois par backend, qui pour chaque service Splynx existant crée un binding PPP local avec le login et le mot de passe chiffré, **sans rotation forcée** ni écriture vers Splynx,
**So that** ADR-002 (import passif) est opérationnel et la rupture PPPoE massive du parc actif est évitée (clôt OQ-15a).

**Référence architecture** : ADR-002 (reprise PPP legacy).

**Acceptance Criteria** :
- **Given** un tenant Splynx avec 10 000 services existants, **When** le job tourne pour ce backend, **Then** 10 000 bindings PPP sont créés avec `source='legacy'`,
- **And** un événement d'audit `ppp.imported.legacy` est émis (1 par binding ; un correlation_id par batch),
- **And** **aucun appel d'écriture** vers Splynx n'est émis (pas de PUT, pas de rotation),
- **And** un compteur de PPP legacy actifs par backend est exposé (consommable Chantier 4),
- **And** un re-run du job est idempotent : pas de duplicate créé.

---

### Story E2.S9 — Snapshot baseline S3 WORM des PPP legacy

**As a** ops migration,
**I want** qu'à la fin de l'import legacy, un snapshot chiffré complet des PPP legacy soit archivé sur S3 avec Object Lock WORM (rétention 5 ans),
**So that** un rollback de secours est possible si la bascule de propriété PPP révèle un problème non anticipé (ADR-002 — reprise PPP legacy §4).

**Référence architecture** : ADR-002 (freeze baseline).

**Acceptance Criteria** :
- **Given** import legacy terminé, **When** le job snapshot tourne, **Then** un fichier chiffré est uploadé sur S3 avec Object Lock COMPLIANCE 5 ans,
- **And** une vérification post-upload : tentative DELETE est rejetée (403),
- **And** un événement d'audit `ppp.baseline.snapshotted` est émis (backend, compte, URI S3).

---

### Story E2.S10 — Dashboard kanban « PPP legacy vs Odoo-generated »

**As a** ops migration,
**I want** un dashboard kanban Odoo qui pour chaque backend affiche les compteurs `legacy_active`, `odoo_generated`, et leur ratio,
**So that** je suis la décroissance attendue du parc legacy au fil du temps (rotations volontaires).


**Acceptance Criteria** :
- **Given** un backend avec 8 000 legacy + 2 000 Odoo-generated, **Then** le dashboard affiche `legacy: 80 %, odoo: 20 %`,
- **And** un historique 30 jours est visible (graphique),
- **And** la métrique correspondante est exposée (consommable Chantier 4).

---

### Story E2.S11 — Actions techniques `suspend` / `reactivate` / `terminate`

**As a** opérateur,
**I want** des actions techniques qui exécutent côté Splynx les opérations block / unblock / terminate via le client HTTP, en mode asynchrone,
**So that** les opérations techniques sont pilotées depuis Odoo et la cohérence d'état Odoo / Splynx converge à la prochaine sync.


**Acceptance Criteria** :
- **Given** un service `active`, **When** un opérateur Contentieux invoque l'action `suspend`, **Then** un job est enqueué (canal critique cf. fairness §5.1 brief commun), l'appel Splynx est émis,
- **And** le statut côté binding Odoo passe à `blocked` après confirmation,
- **And** un événement d'audit `suspension.executed` est émis (correlation_id propagé end-to-end),
- **And** la divergence Odoo / Splynx est vérifiée nulle à T+10 s.

---

### Story E2.S12 — Gating des actions techniques par rôle

**As a** admin fonctionnel,
**I want** que la matrice matrice de gating par rôle dans la SPEC soit appliquée server-side sur les actions techniques (Commercial : aucune ; Administratif : reactivate seulement ; Contentieux et au-dessus : suspend + reactivate ; terminate : Direction Groupe + Administrateur Odoo),
**So that** la matrice de gating est respectée server-side, sans bypass possible via JSON-RPC.


**Acceptance Criteria** :
- **Given** un user Commercial, **When** il tente l'action `suspend` via JSON-RPC, **Then** `AccessError` levé, événement d'audit `role.access.denied` émis,
- **Given** un user Administratif, **When** il tente `suspend`, **Then** `AccessError`,
- **When** il tente `reactivate`, **Then** OK et action exécutée.

---

### Story E2.S13 — Détection asymétrie blocage Splynx + alerte

**As a** ops,
**I want** que si un blocage est détecté côté Splynx sans qu'un job suspension Odoo n'ait été émis dans les 60 minutes précédentes, une alerte soit levée,
**So that** la convention (« toujours suspend/reactivate via Odoo ») est surveillée et le runbook R7 déclenché.


**Acceptance Criteria** :
- **Given** un service synchronisé, **When** Splynx passe en `blocked` sans qu'un job Odoo n'ait initié l'action, **Then** une alerte `manual_block_detected` est levée (notification ops),
- **And** un événement d'audit `splynx.manual_block.detected` est émis,
- **And** le runbook 01 (asymétrie blocage) est référencé dans la notification.

---

### Story E2.S14 — Runbooks 01 (asymétrie blocage) + 04 (import legacy)

**As a** ops,
**I want** deux runbooks markdown détaillés : asymétrie blocage Splynx et procédure d'import legacy PPP (ADR-002 — reprise PPP legacy),
**So that** un on-call non familier peut intervenir en 15 minutes sans réveiller l'architecte.

**Couvre** : convention runbooks (AR-20).

**Acceptance Criteria** :
- **Given** les deux runbooks committed dans le dossier `scripts/runbook/`, **Then** chacun couvre : contexte, symptômes, diagnostic, procédure, rollback, post-mortem checklist,
- **And** un dry-run par un dev non auteur valide la procédure (signature `validated_by` dans le runbook).

---

## 5. Tests d'acceptation Chantier 2

🟡 **À recevoir avant démarrage** : tests red-phase ATDD couvrant les 6 critères d'acceptance globale (§3) et chaque AC story. On vise la couverture suivante :

| Catégorie | Stories sources |
|---|---|
| Storage PPP chiffré at-rest | S1 |
| Générateur PPP (entropie, unicité) | S2 |
| Pre-check anti-collision | S3 |
| Génération + propagation 1 round-trip | S4 |
| Wizard rotation gated + audit | S5 |
| Wizard révélation gated + audit avant affichage | S6 |
| Matrice de visibilité par rôle | S7 |
| Import legacy passif idempotent | S8 |
| Snapshot WORM rollback | S9 |
| Dashboard PPP legacy vs Odoo-generated | S10 |
| Actions techniques + cohérence Odoo/Splynx | S11 |
| Gating actions par rôle | S12 |
| Détection asymétrie blocage + alerte | S13 |
| Runbooks validés | S14 |

---

Et voilà pour le Chantier 2. Bon courage à l'équipe !
