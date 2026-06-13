# AXIA ISP Management Suite

Connecteur bidirectionnel **Odoo 18 LTS ↔ Splynx** pour 4 marques télécom (Marque A, Marque B, Marque C, Marque D). Odoo devient la plateforme métier maîtresse — commercial, facturation, règles de suspension, calendaires multi-pays, audit 5 ans. Splynx reste l'exécutant technique réseau (PPPoE, RADIUS, monitoring).

---

## Par où commencer

Lisez dans cet ordre :

1. **`brief-main.md`** — Conventions transverses, stack figée, onboarding développeur, Definition of Done, procédure d'amendement architecture, ressources & contacts. **À lire en premier**, et à garder en référence permanente pour tous les chantiers.

2. **`briefs/brief-axia-isp-chantier-<N>.md`** — Brief du chantier sur lequel vous démarrez. Contient le scope, l'acceptance globale, les stories, et la liste des tests d'acceptation attendus.

3. **`architecture.md`** — Architecture canonique du projet : 10 décisions d'architecture (ADR) figées, scaffold des modules Odoo, patterns d'implémentation, frontières de données, flux d'intégration. Référence permanente.

4. **`specs/`** — Contrat sémantique du projet :
   - `SPEC.md` — capacités métier (CAP-1 à CAP-15), contraintes, non-goals
   - `admin-parameters.md` — paramètres métier éditables sans code
   - `field-mappings.md` — matrice des correspondances de champs Odoo ↔ Splynx
   - `service-states.md` — machine d'états des statuts contractuels
   - `sync-ownership.md` — matrice de propriété par champ (Odoo-owned vs Splynx-owned)
   - `acceptance-scenarios.md` — scénarios d'acceptation E2E (Given/When/Then)
   - `glossary.md` — glossaire métier ISP

---

## Structure du repo

```
ax-odoo/
├── README.md             # Ce document
├── brief-main.md         # Brief commun (à lire en premier)
├── architecture.md       # Architecture canonique (10 ADR figés)
├── briefs/               # Briefs chantier par chantier
│   ├── brief-axia-isp-chantier-0.md
│   ├── brief-axia-isp-chantier-1.md
│   ├── brief-axia-isp-chantier-2.md
│   ├── brief-axia-isp-chantier-3.md
│   └── brief-axia-isp-chantier-4.md
└── specs/                # SPEC + 6 companions (contrat sémantique)
    ├── SPEC.md
    ├── admin-parameters.md
    ├── field-mappings.md
    ├── service-states.md
    ├── sync-ownership.md
    ├── acceptance-scenarios.md
    └── glossary.md
```

---

## Méthode de livraison

Le projet est découpé en **5 chantiers livrés séquentiellement** :

| Chantier                          | Objet                                  | Stories |
| --------------------------------- | -------------------------------------- | ------- |
| 0 — Fondations (Sprint 0)         | Bootstrap, audit, RBAC, POC Splynx, CI | 10      |
| 1 — Synchronisation Odoo ↔ Splynx | Connecteur bidirectionnel              | 23      |
| 2 — PPP & actions techniques      | Gestion PPP + suspend/réactiver        | 14      |
| 3 — Workflow impayés & billing    | Automatisation impayés multi-pays      | 20      |
| 4 — Production-ready ops          | Observabilité + archivage + RGPD       | 15      |


**Tests d'acceptation** : pour chaque chantier, une suite de tests **red-phase** (initialement rouges) vous est transmise en même temps que le brief. Votre travail : implémenter le code pour que ces tests passent au vert sans modifier leurs assertions. Cette suite constitue le contrat dur de livraison.

---

## Stack imposée

- Odoo Community 18 LTS + PostgreSQL 15 + Redis 7
- Modules OCA : `connector`, `queue_job`, `keychain`
- Client HTTP sortant : `httpx` (wrapper minimal custom)
- Calendaires : `python-holidays`
- Tests : `pytest-odoo`
- Déploiement : Docker + Docker Compose (on-premise)
- Observabilité (Chantier 4) : Prometheus + Loki + Sentry
- Stockage archivage immuable (Chantier 4) : S3 + Object Lock

Détails complets et versions à pinner dans `brief-main.md` §3 et `architecture.md`.

