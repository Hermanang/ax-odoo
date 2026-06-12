# Glossaire — AXIA ISP Management Suite

Termes ISP / télécom et concepts produit utilisés dans `SPEC.md` et ses companions.

## Acteurs et systèmes

- **Opérateur** — l'entreprise cliente du projet ; fournisseur d'accès Internet (FAI) ou opérateur télécom multi-services.
- **Abonné / client** — personne physique ou morale ayant un contrat avec l'opérateur.
- **Odoo** — plateforme métier maîtresse du projet : commercial, CRM, facturation côté client, règles métier, audit, UI de pilotage.
- **Splynx** — logiciel ISP non-extensible déjà en place ; gère réseau, RADIUS, monitoring, exécution technique des coupures/raccordements. Reste source de vérité pour l'état technique.
- **Connecteur** — module Odoo qui traduit les actions et synchronisations entre Odoo et Splynx (API REST côté Splynx).

## Réseau et services

- **PPP** (Point-to-Point Protocol) — protocole d'authentification couramment utilisé pour les connexions FAI. Un service Internet possède typiquement un login et un mot de passe PPP.
- **RADIUS** — protocole d'authentification/autorisation/comptabilité que Splynx utilise pour valider les sessions abonnés.
- **NAS** (Network Access Server) — équipement réseau qui termine les sessions abonnés (BAS, BNG, OLT). Splynx remonte le NAS utilisé pour une session active.
- **IPv4 / IPv6** — adresses IP affectées à un abonné, statiques ou dynamiques, lues depuis Splynx.
- **IP statique** — adresse IP fixe attribuée durablement à un abonné, configurée dans la fiche service.
- **Uptime / durée de connexion** — temps écoulé depuis le début de la session abonné en cours, fourni par Splynx.
- **Online / offline** — état de session courant de l'abonné côté Splynx.
- **IPTV** — service de télévision sur IP.
- **VoIP** (Voice over IP) — service de téléphonie sur IP.
- **Service mobile** — abonnement mobile éventuel ; le connecteur peut être amené à le piloter via Splynx ou un back-end équivalent.

## Cycle de vie commercial et facturation

- **Sync / synchronisation** — propagation d'une modification d'un système à l'autre. Bidirectionnelle dans ce projet (Odoo → Splynx et Splynx → Odoo).
- **Détection de doublon** — recherche d'un client existant avant création, sur nom, prénom, email, téléphone, n° client, identifiant externe.
- **Facture impayée / en retard** — facture dont la date d'échéance est dépassée selon le statut comptable Odoo. Déclencheur du workflow d'impayés.
- **Période de grâce** — délai paramétrable accordé après l'échéance avant déclenchement effectif de la suspension.
- **Délai avant suspension** — temps total écoulé entre l'événement déclencheur (impayé détecté) et l'exécution effective de la coupure ; combine grâce et règles calendaires.
- **Report manuel** — action explicite d'un opérateur repoussant la suspension d'un client donné à une date future, indépendamment des règles globales.

## Suspension, réactivation, calendrier

- **Suspension** — coupure technique du service côté Splynx ; l'abonné perd l'accès mais reste enregistré et facturable selon la politique.
- **Réactivation** — remise en service après suspension (paiement, décision opérateur, expiration d'une cause). Autorisée 24/7.
- **Résiliation** — fin du contrat ; le service et le client sont marqués résiliés dans les deux systèmes.
- **Suspension immédiate** — coupure exécutée dès détection de la condition, sans grâce additionnelle.
- **Suspension différée** — coupure planifiée à une date/heure future, par rapport à un événement déclencheur.
- **Suspension après période de grâce** — coupure exécutée après écoulement d'un délai paramétré post-échéance.
- **Calendrier d'exécution** — ensemble de règles définissant quels jours et plages horaires autorisent ou interdisent les actions de suspension, par pays.
- **Jour ouvré** — jour de la semaine hors week-end et hors jours fériés du pays du client.
- **Jour férié** — jour défini comme férié dans le calendrier du pays du client.

## Données et opérations

- **Identifiant externe** — clé stable utilisée pour faire correspondre un client Odoo à un client Splynx (et inversement).
- **Source de vérité** — système qui fait foi pour un champ donné. Odoo : commercial / décisions métier. Splynx : état technique réseau.
- **No-op sync** — cycle de synchronisation qui ne déclenche aucun appel d'écriture parce qu'aucune donnée pertinente n'a changé.
- **Workflow d'impayés** — enchaînement automatique détection → règles → décision → exécution → propagation → réactivation décrit par CAP-13.
- **Scheduler** — composant Odoo (cron) qui évalue périodiquement les conditions d'impayés et déclenche les actions dues.
- **Historique / journal d'audit** — trace horodatée et corrélable des syncs, décisions et actions techniques exécutées (CAP-15).
