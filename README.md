# AXIA ISP Management Suite

Connecteur bidirectionnel **Odoo 18 LTS ↔ Splynx** pour 4 marques télécom (XIWO, WEELAX, COQLA, GLOBALGRID). Odoo devient la plateforme métier maîtresse (commercial, facturation, règles métier, audit) ; Splynx reste l'exécutant technique réseau (PPPoE, RADIUS, monitoring).

**Statut** : handoff-ready pour Chantier 0. Reste à compléter les `[TBD - Kelvin]` du brief commun (repo URL, contacts, accès Splynx) avant remise au prestataire de développement externe.

---

## Arborescence

```
splynx-odoo/
├── _bmad/                        # Configuration BMad Method (skills, scripts)
├── _bmad-output/                 # 🎯 Tous les artefacts produits (livrables + planning)
│   ├── planning-artifacts/        # CDC, PRD, architecture, epics, readiness report
│   ├── specs/                     # SPEC + companions (contrat sémantique)
│   ├── implementation-artifacts/  # Briefs handoff par chantier (le package à livrer)
│   └── test-artifacts/            # Tests d'acceptation ATDD red-phase
├── inputs/                       # Fichiers source internes (PDFs, spécifications client)
├── docs/ + docs-projet/          # Documentation projet
├── *.pdf                         # PDFs sources internes (CDC consolidés)
└── README.md                     # (ce document)
```

---

## Pour le prestataire de développement externe

Le **package handoff** à transmettre se compose des fichiers suivants (PAS le repo entier — uniquement ce qui suit) :

```
📦 Package handoff Chantier N
├── _bmad-output/implementation-artifacts/briefs/
│   ├── brief-axia-isp-commun.md             # À lire en premier
│   └── brief-axia-isp-chantier-N.md         # Chantier en cours uniquement
├── _bmad-output/planning-artifacts/architecture.md   # Architecture canonique (10 ADR)
├── _bmad-output/specs/spec-axia-isp/                 # Contrat sémantique
│   ├── SPEC.md
│   ├── admin-parameters.md
│   ├── field-mappings.md
│   ├── service-states.md
│   ├── sync-ownership.md
│   ├── acceptance-scenarios.md
│   └── glossary.md
└── _bmad-output/test-artifacts/atdd-chantier-N/      # Tests red-phase
```

**Documents à NE PAS transmettre** (internes) : `inputs/`, PDFs racine, `_bmad-output/planning-artifacts/cahier-des-charges/`, `_bmad-output/planning-artifacts/prds/`, `_bmad-output/planning-artifacts/research/`.

Chaque chantier est livré séquentiellement : Chantier 0 → 1 → 2 → 3 → 4. Le brief Chantier N+1 + ses tests ATDD sont remis après acceptation et sign-off du Chantier N.

---

## État du projet

### Phase 1 — Planning (terminée)

| Artefact | Statut | Localisation |
|---|---|---|
| Cahier des charges v2 | ✅ | `_bmad-output/planning-artifacts/cahier-des-charges/cdc-axia-isp-2026-05-27-v2.md` |
| PRD v2 + addendum + reviews | ✅ | `_bmad-output/planning-artifacts/prds/prd-axia-isp-2026-05-27/` |
| Architecture (10 ADR, 7 modules) | ✅ figée | `_bmad-output/planning-artifacts/architecture.md` |
| SPEC + 6 companions | ✅ | `_bmad-output/specs/spec-axia-isp/` |
| Epics + 82 stories | ✅ | `_bmad-output/planning-artifacts/epics.md` |
| Readiness Report | ✅ verdict READY | `_bmad-output/planning-artifacts/implementation-readiness-report-2026-06-11.md` |

### Phase 2 — Handoff (en cours)

| Artefact | Statut | Localisation |
|---|---|---|
| Brief commun + 5 briefs chantier | ✅ | `_bmad-output/implementation-artifacts/briefs/` |
| Tests d'acceptation ATDD Chantier 0 | ✅ 76 tests | `_bmad-output/test-artifacts/atdd-chantier-0/` |
| Tests d'acceptation ATDD Chantier 1 | ✅ 65 tests | `_bmad-output/test-artifacts/atdd-chantier-1/` |
| Tests d'acceptation ATDD Chantier 2 | ✅ 25 tests | `_bmad-output/test-artifacts/atdd-chantier-2/` |
| Tests d'acceptation ATDD Chantiers 3+4 | ⏸️ À produire | — |
| Compléter `[TBD - Kelvin]` brief commun §8 | 🟡 À faire | repo URL, contacts, accès Splynx |
| Préparer canal sécurisé pour tokens Splynx | 🟡 À faire | Bitwarden / 1Password share |

### Phase 3 — Implémentation (à venir)

Démarre après remise du package Chantier 0 au prestataire externe. Cadence : 1 chantier à la fois, sign-off explicite Kelvin avant passage au suivant.

| Chantier | Objet | Stories | Effort estimé |
|---|---|---|---|
| 0 — Fondations | Bootstrap + audit + RBAC + POC Splynx + CI | 10 | ~35 j dev |
| 1 — Sync Odoo ↔ Splynx | Connecteur bidirectionnel | 23 | ~76 j dev |
| 2 — PPP & actions techniques | Gestion PPP + suspend/réactiver | 14 | ~38 j dev |
| 3 — Workflow impayés | Automatisation impayés multi-pays | 20 | ~55 j dev |
| 4 — Production-ready ops | Observabilité + RGPD + WORM | 15 | ~42 j dev |

---

## Conventions internes

- **Stack figée** : Odoo 18 LTS + PostgreSQL 15 + Redis + OCA `connector`/`queue_job`/`keychain` + Docker on-premise.
- **Multi-company natif** (4 marques étanches), code agnostique au mode pour bascule multi-DB triviale si DPO l'impose.
- **Audit 5 ans glissants** avec archivage S3 WORM (Object Lock 5 ans).
- **PPP généré par Odoo** (CDC v2 inverse v1) — Splynx applique.
- **Aucune suppression physique** côté Splynx (résiliation = `terminated`).

---

## Travailler sur ce projet avec un agent IA

Le projet est instrumenté pour [BMad Method](https://docs.bmad-method.org/) et utilisable via Claude Code, Cursor ou équivalent. Les agents persistants disponibles :

- **Winston** (System Architect) — `bmad-agent-architect`
- **Paige** (Technical Writer) — `bmad-agent-tech-writer`
- **John** (Product Manager) — `bmad-agent-pm`
- **Mary** (Business Analyst) — `bmad-agent-analyst`
- **Amelia** (Developer) — `bmad-agent-dev`
- **Sally** (UX Designer) — `bmad-agent-ux-designer`
- **Murat** (Test Architect) — `bmad-tea`

Mémoire persistante des décisions et conventions dans `~/.claude/projects/-Users-mac-Documents-projects-splynx-odoo/memory/`.

---

**Propriétaire projet** : Kelvin
**Date de dernière mise à jour** : 2026-06-12
