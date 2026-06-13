---
title: Scénarios d'acceptation testables — Flux clés AXIA ISP
project: AXIA ISP Management Suite
audience: Équipe développement externe
status: Contrat sémantique — référencé par les briefs chantier
version: 1.0
date: 2026-06-12
---

# Scénarios d'acceptation testables

> 📖 **Comment lire ce document.** Ce companion liste les **scénarios d'acceptation E2E** que les tests des chantiers doivent reproduire. Chaque scénario est exprimé en Given/When/Then et constitue un **contrat comportemental** : le code livré doit faire passer ces scénarios verts (mais le **comment** vous appartient — naming d'implémentation libre, cf. brief commun §5).
>
> Ces scénarios sont référencés depuis :
> - Brief Chantier 1 §3 critère 8 + Story E1.S23 (scénarios §15.1 + §15.6)
> - Brief Chantier 2 §3 critère 6 (scénario §15.4)
> - Brief Chantier 3 §3 critères 1, 2, 3, 5 + Stories E3.S17 à E3.S20 (scénarios §15.2, §15.3, §15.4, §15.5, §15.7)

## Sommaire

1. [Création client conditionnelle (5 sous-scénarios)](#1-création-client-conditionnelle)
2. [Workflow d'impayés bout-en-bout — Mali grace_period](#2-workflow-dimpayés-bout-en-bout--mali-grace_period)
3. [Suspension avec respect calendrier multi-pays — Guadeloupe 27 mai + week-end](#3-suspension-calendrier-multi-pays)
4. [Réactivation 24/7](#4-réactivation-247)
5. [Report manuel par client](#5-report-manuel-par-client)
6. [Conflit de sync bidirectionnel (3 sous-scénarios)](#6-conflit-de-sync-bidirectionnel)
7. [Cohabitation multi-société](#7-cohabitation-multi-société)

---

## 1. Création client conditionnelle

### 1.1 Sous-scénario — Prospect pur (Splynx non-éligible)

**Given** un `res.partner` est créé dans Odoo sans commande ISP confirmée ni souscription / contrat actif portant un produit avec `is_isp_service = True`,
**When** le prochain cycle du connecteur s'exécute,
**Then** **aucune ligne de binding** n'est créée,
**And** **aucun appel de création** n'est émis vers Splynx,
**And** le champ `splynx_status` affiche `not_eligible` (couleur neutre),
**And** aucun événement de création n'est journalisé pour ce partner.

### 1.2 Sous-scénario — Vente confirmée d'un service ISP (Splynx éligible)

**Given** un `res.partner` Odoo,
**When** une commande de vente portant une ligne avec `is_isp_service = True` passe en état confirmé,
**Then** le champ `splynx_status` passe à `eligible_pending` (couleur attention),
**And** un bandeau d'avertissement « Ce client va être créé dans Splynx au prochain run » apparaît sur la vue formulaire,
**When** le prochain run du connecteur s'exécute,
**Then** une ligne de binding est créée en transaction,
**And** un client correspondant apparaît dans Splynx avec les champs mappés via le Mapper v1 (vérifiable par un `GET /api/2.0/admin/customers/customer/{id}` côté tenant),
**And** le champ `splynx_status` passe à `synced` (couleur OK) avec affichage de l'ID Splynx,
**And** un événement d'audit `customer.created` est enregistré (source Odoo, direction O→S, succès, avec `correlation_id`).

### 1.3 Sous-scénario — Bouton manuel « Forcer création »

**Given** un `res.partner` Odoo en état `not_eligible` (cas exceptionnel : migration legacy),
**And** l'opérateur appartient au groupe administrateur Odoo (rôle 7 — cf. SPEC),
**When** il clique sur « Forcer création Splynx » et renseigne un motif obligatoire (≥ 10 caractères),
**Then** une ligne de binding est créée immédiatement,
**And** un événement d'audit `splynx.force_creation` (ou `customer.force_creation`) est journalisé avec auteur, motif, horodatage,
**And** le partner passe à `synced` après confirmation Splynx.

### 1.4 Sous-scénario — Tentative de création par utilisateur non-admin

**Given** un `res.partner` Odoo en état `not_eligible`,
**And** un utilisateur **n'appartenant pas** au groupe administrateur Odoo,
**When** il tente d'accéder au bouton « Forcer création »,
**Then** le bouton est invisible / non-cliquable,
**And** une tentative d'appel via JSON-RPC direct lève une `AccessError`,
**And** aucun binding ne peut être créé.

### 1.5 Sous-scénario — Rapport divergence quotidien

**Given** un partner avec une commande ISP confirmée et **sans** binding,
**When** le cron quotidien tourne,
**Then** le partner apparaît dans le rapport « Partners éligibles non synchronisés »,
**And** un email d'alerte est envoyé à l'opérateur configuré si au moins un cas est détecté.

---

## 2. Workflow d'impayés bout-en-bout — Mali grace_period

**Scénario nominal — Mali, mode `grace_period` 10 jours, semaine ouvrée**

**Given** un client `C_ML` (société Mali, calendrier Mali, mode `grace_period`, nombre de jours de grâce = 10),
**Given** une facture `F_C_ML` bascule en `late` un mardi à 14h00,
**When** le scheduler tourne,
**Then** un événement `invoice.overdue.detected` est émis dans les 15 minutes,
**And** la suspension est planifiée à J+10 (jour ouvré, pas un jour férié au Mali),
**And** entre J et J+10 les notifications email + SMS sont déclenchées selon configuration (déclenchement par AXIA, routage hors scope),
**When** à J+10 le client n'a toujours pas payé,
**Then** la suspension est exécutée sur le canal critique de suspension (haute priorité),
**And** le statut Splynx du client passe à `blocked` dans les 5 minutes p95,
**And** le statut Odoo du service passe à « Suspendu pour non-paiement »,
**And** les événements d'audit `suspension.executed` et `field.overwrite` (statut) sont journalisés avec un `correlation_id` commun,
**When** le client paie le lendemain,
**Then** un événement `payment.received` est émis,
**And** la réactivation est exécutée dans les 5 minutes p95 sur le canal critique de paiement,
**And** le statut Splynx passe à `Activated`,
**And** un événement `reactivation.executed` (ou `activation.executed`) est journalisé.

---

## 3. Suspension calendrier multi-pays

### 3.1 Sous-scénario — Guadeloupe 27 mai (abolition de l'esclavage, férié local)

**Given** un client `C_GP` rattaché à la subdivision Guadeloupe (`state_id = 971`), calendrier Guadeloupe, paramètre `block_on_holidays = True`,
**Given** une facture impayée dont la suspension calculée tombe le 27 mai 2026,
**When** le scheduler tourne le 27 mai,
**Then** la suspension n'est **pas exécutée** ce jour-là,
**And** elle est reprogrammée au prochain jour autorisé (28 mai si non férié et non week-end),
**And** un événement d'audit `suspension.postponed.calendar` est journalisé avec :
- `motif = holiday`
- `country = 971`
- `date_initiale = 2026-05-27`
- `date_reprogrammée = 2026-05-28`
- `nom_du_férié = "Abolition de l'esclavage"`

### 3.2 Sous-scénario — Week-end (samedi bloqué)

**Given** un client `C_FR` rattaché à la France métropolitaine, paramètre `block_on_saturday = True`,
**Given** une facture impayée dont la suspension calculée tombe un samedi,
**When** le scheduler tourne ce samedi,
**Then** la suspension est reportée au lundi suivant (ou prochain jour autorisé selon `block_on_holidays`),
**And** un événement d'audit `suspension.postponed.calendar` est journalisé avec `motif = saturday`.

---

## 4. Réactivation 24/7

### 4.1 Sous-scénario — Réactivation samedi 22h00 (configuration par défaut)

**Given** un client `C_FR` suspendu,
**And** les paramètres par défaut `activation_allowed_weekend = True` et `activation_on_holidays = True`,
**When** un paiement est encaissé un samedi à 22h00,
**Then** un événement `payment.received` est émis,
**And** la réactivation est exécutée dans la fenêtre ≤ 5 minutes p95 sur le canal critique de paiement (haute priorité),
**And** le statut Splynx passe à `Activated` (vérifiable par un GET côté tenant),
**And** un événement `reactivation.executed` (ou `activation.executed`) est journalisé avec horodatage entre samedi 22h00 et 22h05.

### 4.2 Sous-scénario — Désactivation explicite des réactivations week-end

**Given** le paramètre `activation_allowed_weekend = False`,
**When** un paiement est encaissé un samedi,
**Then** la réactivation est reportée au prochain jour ouvré (par exemple lundi à 08h00),
**And** un événement d'audit `activation.postponed.calendar` est journalisé avec le motif.

---

## 5. Report manuel par client

**Given** un client `C_FR` avec une facture impayée,
**And** le champ `cutoff_postponed_until` positionné à la date `2026-06-15`,
**When** le scheduler tourne entre aujourd'hui et le 14 juin 2026,
**Then** la décision est marquée « report manuel actif »,
**And** **aucune suspension** n'est exécutée,
**And** un événement `invoice.overdue.detected` est tout de même émis à chaque cycle (visibilité opérateur),
**When** le 16 juin 2026 le scheduler tourne,
**Then** un événement `cutoff.postponed.expired` est émis,
**And** l'évaluation normale des règles reprend (délai, grâce, calendrier),
**And** la suspension est planifiée selon les règles normales si la facture est toujours impayée.

---

## 6. Conflit de sync bidirectionnel

### 6.1 Sous-scénario — Champ Odoo-owned divergent

**Given** un client `C` avec son `email` synchronisé entre Odoo et Splynx,
**When** un agent réseau modifie manuellement l'`email` côté Splynx,
**And** au cycle de sync suivant AXIA détecte la divergence,
**Then** la valeur Odoo écrase la valeur Splynx (cf. matrice d'ownership dans `sync-ownership.md`),
**And** un événement d'audit `field.overwrite` est journalisé avec :
- `field = email`
- `source = odoo`
- `before_splynx = <ancienne valeur côté Splynx>`
- `after_odoo = <valeur Odoo poussée>`
- horodatage

### 6.2 Sous-scénario — Champ Splynx-owned divergent

**Given** un client `C` avec un champ Splynx-owned `current_ipv4` exposé en lecture seule dans Odoo,
**When** une tentative d'écriture manuelle est faite côté Odoo (via API directe contournant l'UI),
**And** au cycle de sync suivant AXIA détecte la divergence,
**Then** la valeur Splynx écrase la valeur Odoo,
**And** un événement d'audit `field.overwrite` est journalisé avec :
- `field = current_ipv4`
- `source = splynx`

### 6.3 Sous-scénario — Aucune divergence

**Given** un client `C` non modifié,
**When** le cycle de sync s'exécute,
**Then** **aucun appel d'écriture** vers Splynx n'est émis (no-op strict),
**And** aucun événement `field.overwrite` n'est journalisé.

---

## 7. Cohabitation multi-société

> Scénario aligné sur le contexte AXIA réel : opérateur exploitant **4 marques (Marque A, Marque B, Marque C, Marque D)** portées par des sociétés distinctes sous propriétaire commun. Le scénario ci-dessous illustre la cohabitation sur deux sociétés représentatives ; le principe d'étanchéité s'applique identiquement aux quatre.

**Given** une instance Odoo avec deux sociétés :
- `S_FR_marqueA` : calendrier France métropolitaine, `block_on_saturday = True`, mode `grace_period`, nombre de jours de grâce = 15
- `S_SN_marqueB` : calendrier Sénégal, `block_on_saturday = False`, mode `immediate`

**Given** chaque société a son propre backend Splynx (`B_FR`, `B_SN`),

### 7.1 Sous-scénario — Suspension paramétrée par société

**When** un client `C_FR` (société `S_FR_marqueA`) a une facture impayée un samedi,
**Then** sa suspension est reportée au lundi (selon paramètres de `S_FR_marqueA`),
**And** elle est exécutée sur `B_FR` exclusivement (pas de fuite vers `B_SN`).

**When** un client `C_SN` (société `S_SN_marqueB`) a une facture impayée le même samedi,
**Then** sa suspension est exécutée immédiatement (selon paramètres de `S_SN_marqueB`),
**And** elle est exécutée sur `B_SN` exclusivement.

### 7.2 Sous-scénario — Isolation des données par utilisateur

**Given** un utilisateur Odoo rattaché uniquement à la société `S_FR_marqueA`,
**When** il consulte la liste des clients,
**Then** **aucun client de `S_SN_marqueB`** n'apparaît,
**And** aucun événement d'audit de `S_SN_marqueB` n'apparaît dans son journal,
**And** il ne peut pas accéder à la configuration du backend `B_SN` (lecture ni écriture).

---

## Annexe — Correspondance scénarios ↔ tests d'acceptation

Les tests d'acceptation ATDD livrés en parallèle de chaque brief chantier couvrent ces scénarios :

| Scénario | Test E2E livré dans |
|---|---|
| §1 Création client conditionnelle | `atdd-chantier-1/test_e1_s23_e2e_acceptance_scenarios.py` |
| §2 Mali grace_period | `atdd-chantier-3/test_e3_s18_*` (à produire — Task #3 en cours) |
| §3 Guadeloupe + samedi | `atdd-chantier-3/test_e3_s17_*` |
| §4 Réactivation 24/7 | `atdd-chantier-2/` + `atdd-chantier-3/test_e3_s20_*` |
| §5 Report manuel | `atdd-chantier-3/test_e3_s20_*` |
| §6 Conflit sync | `atdd-chantier-1/test_e1_s23_e2e_acceptance_scenarios.py` |
| §7 Multi-société | `atdd-chantier-3/test_e3_s19_*` |

---

**Fin du document.** Toute ambigüité sur un scénario doit être signalée en sync hebdo avant écriture du test E2E correspondant — pas d'interprétation silencieuse.
