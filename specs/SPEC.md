---
id: SPEC-axia-isp
companions:
  - glossary.md
  - service-states.md
  - field-mappings.md
  - admin-parameters.md
  - sync-ownership.md
  - acceptance-scenarios.md
---

> **Contrat sémantique.** Ce SPEC et les fichiers listés dans `companions:` constituent le contrat complet de ce qui doit être construit, testé et validé.

# AXIA ISP Management Suite — Plateforme métier Odoo ↔ connecteur Splynx

## Why

Un opérateur télécom / FAI utilise Splynx pour ses opérations techniques réseau (clients, services, supervision, exécution des coupures et raccordements), mais Splynx n'est pas extensible et présente deux limites bloquantes : (1) il coupe un abonné automatiquement à l'échéance, sans respect des exigences réglementaires européennes (notification préalable, fenêtre de grâce, interdiction de coupures les jours non ouvrés), et (2) il ne centralise pas la vision commerciale et métier. Le projet vise à faire d'**Odoo la plateforme métier maîtresse** (commercial, facturation côté client, règles de suspension, calendriers, audit, multi-sociétés) et à reléguer Splynx à un **exécutant technique** piloté par connecteur. Splynx continue de faire ce qu'il fait bien (réseau, RADIUS, monitoring), Odoo devient la source de vérité métier et le seul point de pilotage humain. Le contexte qui rend ce travail urgent : conformité européenne et locale, charge support liée aux coupures non maîtrisées, besoin d'industrialiser des règles d'impayés aujourd'hui manuelles, et anticipation d'une montée en charge rapide (objectif scaling 10 000 – 100 000 abonnés, plusieurs pays dont la France métropolitaine, les DOM-TOM, le Mali et le Sénégal).

## Capabilities

- id: CAP-1
  intent: L'opérateur crée ou modifie un client dans Odoo et le système propage les données selon la matrice de propriété définie dans [[sync-ownership]] ; la création initiale dans Splynx n'est déclenchée que si les flags d'éligibilité de la fiche Odoo sont satisfaits (voir [[field-mappings]] §Conditions de création).
  success: Un client créé dans Odoo avec `create_to_radius = True` et un type de connexion sélectionné apparaît dans Splynx avec les champs mappés (voir [[field-mappings]]) ; un champ Odoo-owned modifié des deux côtés voit la valeur Odoo gagner au prochain cycle, et l'écrasement est tracé.

- id: CAP-2
  intent: Avant toute création dans Splynx, le système recherche les doublons (nom, prénom, email, téléphone, n° client, identifiant externe) et bascule en mise à jour si une correspondance est trouvée.
  success: Un flux d'import ou une création manuelle ne produit aucun doublon dans Splynx ; le cas "match trouvé" génère un événement de mise à jour tracé.

- id: CAP-3
  intent: Le système n'émet de synchronisation que lorsque des champs synchronisables ont effectivement changé.
  success: Un cycle de synchronisation programmé sur un client non modifié n'émet aucun appel d'écriture vers Splynx, vérifiable dans les logs.

- id: CAP-4
  intent: L'opérateur consulte et gère depuis Odoo les attributs techniques d'un service Internet (identifiants PPP, mots de passe PPP, type de service, IP statiques, équipements, routeur/modem).
  success: Tout attribut technique présent dans Splynx est consultable dans la fiche service d'Odoo et modifiable selon la stratégie de sync définie pour ce champ.

- id: CAP-5
  intent: Les statuts abonné (voir [[service-states]] pour le mapping Odoo↔Splynx) sont synchronisés ; les décisions métier émises par Odoo (ex. résiliation) se propagent à Splynx **sans suppression physique** de l'enregistrement.
  success: Une résiliation enregistrée dans Odoo entraîne le passage du client dans Splynx au statut `terminated` (l'enregistrement persiste) ; un changement de statut technique côté Splynx remonte dans Odoo.

- id: CAP-6
  intent: Le système expose dans Odoo les informations techniques temps réel issues de Splynx (online/offline, IPv4, IPv6, NAS, uptime, état des services réseau).
  success: La fiche client dans Odoo affiche un état réseau à jour permettant à un agent support de diagnostiquer un incident sans ouvrir Splynx.

- id: CAP-7
  intent: Le système identifie en continu les factures atteignant le statut Odoo `late` (paramétrable via `overdue_invoice_status`) et déclenche le workflow d'impayés (voir CAP-13).
  success: Une facture qui bascule `late` génère un événement traçable qui entre dans le workflow d'impayés à la prochaine exécution du scheduler ; modifier `overdue_invoice_status` vers un autre statut applique le nouveau déclencheur sans dev.

- id: CAP-8
  intent: L'opérateur configure plusieurs modes de suspension (`immediate`, `delayed`, `grace_period`) avec délais, unités et exceptions (voir [[admin-parameters]]).
  success: Une règle "mode `grace_period`, 5 jours de grâce, suspension auto activée" est exécutée à la date et l'heure attendues ; modifier le délai à 10 jours via l'admin Odoo applique le nouveau comportement au prochain cycle, sans livraison de code.

- id: CAP-9
  intent: Le système respecte des règles calendaires (week-ends, jours fériés) interdisant ou autorisant la suspension selon le jour. Les jours fériés sont détectés selon le pays du client (`customer.country_id`) et activés via `block_on_holidays`, `block_on_saturday`, `block_on_sunday` (voir [[admin-parameters]]). Pays cibles v1 : France métropolitaine, DOM-TOM (Guadeloupe, Martinique, Saint-Martin, Guyane), Mali, Sénégal. L'ajout d'un nouveau pays se fait par configuration.
  success: Un impayé dont la suspension calculée tombe un jour férié local pour le pays du client (ex. 27 mai en Guadeloupe) n'entraîne pas de coupure ce jour-là ; l'action est reportée au prochain jour autorisé et le report est tracé.

- id: CAP-10
  intent: Un opérateur peut décaler manuellement la suspension d'un client donné via le champ `cutoff_postponed_until` (date future), indépendamment des règles globales d'impayés.
  success: Un client avec `cutoff_postponed_until = JJ/MM` n'est jamais suspendu avant cette date même si la facture reste impayée ; l'expiration du report relance l'évaluation normale des règles.

- id: CAP-11
  intent: La réactivation d'un service est autorisée à tout moment (24/7, week-ends et jours fériés inclus) sauf si `activation_allowed_weekend` ou `activation_on_holidays` est explicitement désactivé ; le calendrier de suspension ne s'applique pas aux réactivations.
  success: Un paiement reçu un samedi soir avec les paramètres par défaut déclenche la réactivation automatique du service côté Splynx via le connecteur dans la fenêtre de latence définie.

- id: CAP-12
  intent: Odoo déclenche, depuis ses workflows métier, des actions techniques (suspension, réactivation, résiliation) sur les services réseau via le connecteur. Périmètre v1 : **Internet / PPPoE / Radius**. IPTV / VoIP / mobile sont reportés à une version ultérieure (cf. Non-goals).
  success: L'action "résilier" déclenchée sur un abonnement Internet dans Odoo est transmise à Splynx, reçoit une confirmation, et l'état Odoo/Splynx reste cohérent à la prochaine sync.

- id: CAP-13
  intent: Le système enchaîne automatiquement détection d'impayé → évaluation des règles applicables → vérification des délais et exceptions → décision (suspendre ou reporter) → exécution technique → propagation des statuts → réactivation automatique ou manuelle.
  success: Un cycle complet impayé → suspension → paiement → réactivation s'exécute sans intervention manuelle pour un cas conforme aux règles ; chaque étape est tracée et rejouable.

- id: CAP-14
  intent: Tout paramètre métier listé dans [[admin-parameters]] (déclencheur d'impayé, délais, grâce, modes, calendriers week-ends/fériés, autorisations d'activation, etc.) est éditable depuis l'admin Odoo sans modification de code.
  success: Un administrateur fonctionnel modifie un paramètre listé dans [[admin-parameters]] depuis l'UI Odoo, la nouvelle valeur s'applique au prochain cycle du scheduler sans redéploiement ni redémarrage.

- id: CAP-15
  intent: Le système conserve un historique horodaté des opérations de synchronisation, des décisions métier (suspendre/reporter/réactiver), des actions techniques exécutées via le connecteur et des écrasements de conflits de sync. Rétention : 5 ans (alignée sur l'obligation comptable française).
  success: Pour un client donné, un opérateur retrouve depuis Odoo la chronologie complète des syncs, suspensions, réactivations, écrasements de conflits et leurs causes, sur 5 ans glissants.

## Constraints

- Splynx n'est pas extensible ; toute logique métier (règles, calendriers, décisions) vit dans Odoo. Splynx exécute uniquement les actions techniques réseau commandées par le connecteur.
- Aucune suspension automatique ne s'exécute sans respect du délai de grâce paramétré et des règles calendaires (jours fériés/week-ends configurables par pays), en cohérence avec les exigences réglementaires européennes encadrant la coupure d'abonnés.
- Toutes les règles métier listées dans [[admin-parameters]] (délais, grâce, modes, calendriers, autorisations, déclencheur d'impayé) sont éditables sans code depuis l'admin Odoo.
- La réactivation des services reste autorisée 24/7, week-ends et jours fériés inclus par défaut — le calendrier de suspension ne s'applique pas aux actions d'activation/réactivation, sauf désactivation explicite des paramètres dédiés.
- La résolution de conflit bidirectionnel suit un **modèle field-level ownership** documenté dans [[sync-ownership]] : Odoo gagne sur les champs commerciaux et identitaires, Splynx gagne sur les champs techniques temps réel. Toute divergence détectée déclenche une écriture vers le propriétaire et une trace d'écrasement.
- La résiliation côté Splynx se fait **par changement de statut vers `terminated`**, jamais par suppression physique de l'enregistrement client.
- L'architecture doit être **scalable de 10 000 à 100 000 abonnés** sans réarchitecture : sync incrémentale (CAP-3), exécution asynchrone des actions techniques (files de tâches), pas de batch monolithique, paramétrage et catalogue pays extensibles par configuration.
- La plateforme doit fonctionner en **multi-sociétés Odoo natif** : une instance peut héberger plusieurs opérateurs/sociétés sans cross-contamination de données ou de règles.
- L'historique d'audit est conservé **5 ans glissants** minimum (obligation comptable française), couvrant syncs, décisions métier, actions techniques et écrasements de conflits.

## Non-goals

- Remplacer Splynx pour les opérations techniques réseau (NAS, RADIUS, PPP, gestion des équipements actifs).
- Implémenter dans Odoo un moteur de monitoring réseau temps réel ; Odoo se contente d'exposer les données fournies par Splynx.
- Réimplémenter dans Odoo la facturation technique service-bound déjà assurée par Splynx ; Odoo gère le volet commercial et la décision métier, Splynx la facturation technique liée aux services réseau.
- Gérer en propre les équipements télécom au-delà de leur référence dans la fiche service.
- Couvrir IPTV / VoIP / services mobiles en v1 ; le périmètre v1 est **Internet / PPPoE / Radius uniquement**. Les autres services restent une extension pour une version ultérieure.
- Implémenter la passerelle de paiement (Stripe / GoCardless / SEPA) et le fournisseur SMS dans ce projet : ce sont des **projets distincts** menés en parallèle. AXIA ISP Suite consomme et produit des événements Odoo natifs (paiement reçu, notification à envoyer) ; le branchement aux fournisseurs externes est porté par les modules dédiés hors de ce SPEC.
- Concevoir le moteur de notifications client (canaux, templates, fournisseurs) ; AXIA déclenche le *trigger* "client à notifier" et laisse Odoo (et les modules notification existants ou à venir) router le message.

## Success signal

L'opérateur pilote depuis Odoo seul un cycle d'abonné complet (création → service actif → facturation → impayé → notifications déclenchées via Odoo → suspension conforme aux règles → paiement → réactivation), Splynx exécutant les actions techniques via le connecteur. Test concret : un client guadeloupéen avec facture passée au statut `late` le 26 mai a (1) reçu les rappels Odoo configurés, (2) n'est ni suspendu samedi ni dimanche, (3) n'est pas suspendu le 27 mai (férié local Guadeloupe), (4) est suspendu le 28 mai à l'heure configurée si la grâce est expirée, et (5) est réactivé automatiquement dans les minutes qui suivent l'encaissement, peu importe le jour. Second test : un opérateur sénégalais hébergé en multi-société sur la même instance Odoo applique ses propres règles sans impact sur l'opérateur français.

## Assumptions

- Splynx expose une API REST suffisante pour : CRUD client, lecture/écriture services, contrôle de statut (suspend/unsuspend/terminate), lecture du monitoring (online/offline, IP, NAS, uptime). Cf. [[field-mappings]] pour les correspondances concrètes.
- Le scheduler d'évaluation des impayés tourne à une fréquence suffisante (au moins une fois par jour ouvré, idéalement plus) pour garantir l'application des règles dans la fenêtre attendue.
- Les calendriers jours fériés des pays cibles (FR métropolitaine, DOM-TOM, Mali, Sénégal) sont disponibles via une source exploitable (lib publique ou table maintenue) avec mise à jour annuelle au minimum.
- Les fournisseurs externes (passerelle de paiement, fournisseur SMS) exposent dans Odoo des événements consommables par AXIA (paiement reçu, notification envoyée) ; AXIA n'a pas à connaître leurs spécificités.

## Closed Questions (décidées)

- **Définition opérationnelle de "facture en retard"** : statut Odoo `late`, paramétrable via `overdue_invoice_status` (voir [[admin-parameters]] et CAP-7).
- **Notifications client avant suspension** : canaux **email + SMS**, nombre de rappels et délais configurables (J-X paramétrables). Le déclenchement des notifications est porté par AXIA mais le routage (email serveur, SMS fournisseur) est délégué aux modules Odoo dédiés, traités hors de ce SPEC.
- **Pays cibles v1** : **France métropolitaine + DOM-TOM (Guadeloupe, Martinique, Saint-Martin, Guyane avec leurs jours fériés locaux) + Mali + Sénégal**. Architecture extensible : ajout d'un pays = ajout d'une source de calendrier en configuration.
- **Génération des identifiants PPP** : **Odoo génère** les identifiants et mots de passe PPP, **Splynx les applique**. Le mot de passe est stocké chiffré at-rest côté Odoo (cf. [[sync-ownership]]).
- **Résolution de conflit bidirectionnel** : **field-level ownership** (cf. [[sync-ownership]]). Odoo gagne sur les champs commerciaux/identitaires, Splynx gagne sur les champs techniques. Chaque divergence détectée déclenche un écrasement tracé.
- **Périmètre v1 services** : **Internet / PPPoE / Radius uniquement**. IPTV, VoIP, mobile reportés à une version ultérieure (déjà en non-goals).
- **Durée de rétention des historiques** : **5 ans glissants** (obligation comptable française), couvrant syncs, décisions métier, actions techniques et écrasements de conflits.
- **Volumétrie cible** : architecture dimensionnée pour **10 000 – 100 000 abonnés** avec montée en charge attendue rapide. Sync incrémentale obligatoire, exécution asynchrone, pas de batch monolithique.
- **Multi-opérateur** : **multi-sociétés Odoo natif**. Une instance peut héberger plusieurs opérateurs sans cross-contamination.
- **Intégrations annexes** : **aucune intégration externe directe portée par AXIA en v1**. La passerelle de paiement (GoCardless / SEPA / Stripe) et le fournisseur SMS sont des projets distincts menés en parallèle ; AXIA consomme et produit des événements Odoo natifs.

## Open Questions

_Aucune question ouverte côté fonctionnel. Les questions techniques (volumétrie de sync par jour, latence cible bout-en-bout, choix d'implémentation des calendriers fériés multi-pays) sont traitées dans `architecture.md` et les briefs chantier._
