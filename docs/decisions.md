# Journal des décisions

Toute décision structurante prise pendant le POC est consignée ici : choix technique,
contournement retenu, abandon d'une piste, écart par rapport au plan.

Objectif : retrouver plusieurs semaines plus tard pourquoi une chose a été faite d'une
certaine manière, et pouvoir le justifier en synthèse de fin de phase.

Format : une entrée par décision, la plus récente en haut.

---

## AAAA-MM-JJ — Titre de la décision

**Contexte.** Ce qui a amené à se poser la question.

**Options envisagées.**

**Décision retenue.**

**Conséquences.** Ce que cela implique pour la suite, et ce qu'il faudra revérifier.

---

## 2026-09-15 — Hébergement du POC sur serveur dédié Kimsufi KS-5

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
