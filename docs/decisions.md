# Journal des décisions

Toute décision structurante prise pendant le POC est consignée ici : choix technique,
contournement retenu, abandon d'une piste, écart par rapport au plan.

Objectif : retrouver plusieurs semaines plus tard pourquoi une chose a été faite d'une
certaine manière, et pouvoir le justifier en synthèse de fin de phase.

Format : une entrée par décision, la plus récente en haut.

---

## 2026-09-17 — Découpage cible du serveur : une seule VM de production

**Contexte.** Le serveur `debian2018` devient hôte de virtualisation. La question du
découpage à terme des applications existantes se pose dès maintenant, puisqu'elle
conditionne le plan d'adressage et les conventions.

**Contrainte déterminante.** Le GPU NVIDIA ne peut être attribué qu'à une seule
machine à la fois. Dès qu'il est passé à une VM, l'hôte ne l'a plus. Or plusieurs
applications métier consomment les modèles locaux via `ollama` et `litellm`.

**Décision.** L'ensemble des applications de production migre à terme dans une VM
unique (`vm-production`), qui reçoit le GPU. L'hôte ne conserve que le rôle
d'hyperviseur. La VM du POC reste séparée.

**Conséquences.**
- Pas d'arbitrage de découpage à faire : la migration est un déplacement en bloc.
- Le passage du GPU et la migration des conteneurs sont une seule et même opération.
- Le facteur dimensionnant de l'interruption sera le volume des données à déplacer
  (bases PostgreSQL, fichiers déposés, modèles `ollama`), pas le nombre
  d'applications.
- Le plafond mémoire des VM défini dans les conventions (32 Go tant que la
  production tourne sur l'hôte) devra être revu au moment de la migration.

**Hors périmètre du POC.** Ce chantier fera l'objet d'un document de cadrage
distinct, avec inventaire applicatif, ordre de migration et fenêtres.

---

## 2026-09-17 — Hébergement du POC sur le serveur interne, en machine virtuelle

**Contexte.** Ni le VPS-4 ni le serveur dédié KS-5 n'étaient disponibles chez OVH,
et la même rupture touchait les autres hébergeurs du fait de la pénurie de mémoire
de 2026. Le calendrier du POC ne permettait pas d'attendre.

**Options envisagées.** Attendre le réapprovisionnement, changer d'hébergeur
(netcup, Contabo, Leaseweb, Scaleway Dedibox), ou héberger en interne.

**Décision.** Hébergement sur le serveur `debian2018`, dans une machine virtuelle
dédiée. Ressources disponibles largement suffisantes : 20 vCPU, 49 Go de mémoire
libre, 1,3 To d'espace.

**Conséquences.** Coût nul, disponibilité immédiate, snapshots retrouvés. En
contrepartie, l'hôte porte des applications de production, d'où les règles
impératives définies dans les conventions de virtualisation.

---

## AAAA-MM-JJ — Titre de la décision

**Contexte.** Ce qui a amené à se poser la question.

**Options envisagées.**

**Décision retenue.**

**Conséquences.** Ce que cela implique pour la suite, et ce qu'il faudra revérifier.

---

## 2026-09-15 — Hébergement du POC sur serveur dédié Kimsufi KS-5

**Statut : caduque.** Remplacée par la décision du 17/09 (hébergement interne en VM).

**Contexte.** Le VPS-4 initialement retenu n'était pas disponible.

**Décision.** Serveur dédié KS-5 (32 Go, 2 × 450 Go NVMe RAID 1, datacenter France).

**Conséquences.** Pas de snapshot système sur bare metal. Le retour arrière repose sur
la reproductibilité depuis ce dépôt, la sauvegarde des volumes Docker et Backup Agent.
Voir `26-09-15_SI-ERP_Plan-projet-POC_V1.01.md`.

---

## 2026-09-15 — Accès distant par WireGuard

**Contexte.** Le serveur dispose d'une IP publique. La politique interne impose
qu'aucun port ne soit exposé hors du strict nécessaire.

**Options.** WireGuard, ou tunnel SSH avec redirection de ports.

**Décision.** WireGuard. Il n'expose qu'un port UDP silencieux, ne répond pas aux
paquets non authentifiés, et permet de retirer SSH de l'interface publique.

**Conséquences.** Chaque personne devant consulter les environnements doit installer un
client et recevoir une configuration. Procédure à documenter dans `docs/securite.md`.
