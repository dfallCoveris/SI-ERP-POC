# Infrastructure — Conventions de virtualisation

**Version : V1.01**
**Date : 17/09/2026**
**Hôte concerné : `debian2018` — 192.168.75.10**
**Étape B du plan `26-09-16_SI-Infra_Plan-virtualisation_V1.00.md`**

---

## 0. Objet et portée

Ce document définit les règles applicables à **toute machine virtuelle** créée sur
l'hôte `debian2018`. Il s'applique à la première VM comme à la dixième.

Il est écrit avant la création de la première VM, volontairement : des conventions
posées après coup ne sont jamais appliquées rétroactivement.

### Contexte contraignant

L'hôte n'est pas une machine dédiée à la virtualisation. Il exécute simultanément
une vingtaine de conteneurs Docker en production et une pile d'IA locale exploitant
un GPU. **Ces applications passent avant toute VM.** Les règles ci-dessous découlent
de cette contrainte.

---

## 1. Nommage

### Machines virtuelles
`vm-<usage>`, en minuscules, sans accent, sans espace, sans caractère spécial autre
que le tiret.

Exemples : `vm-erp-poc`, `vm-outils-internes`, `vm-ia`.

Le nom doit décrire l'usage, pas la technologie ni le numéro. `vm-erp-poc` se
comprend dans deux ans, `vm-01` non.

### Nom d'hôte interne
Identique au nom de la VM. Une VM nommée `vm-erp-poc` a pour nom d'hôte
`vm-erp-poc`. Aucune exception : un écart entre les deux crée des confusions lors
des diagnostics.

### Disques virtuels
`<nom-vm>.qcow2`, dans `/var/lib/libvirt/images/`.

### Snapshots
`<objet>-<AAAAMMJJ>`, en minuscules.

Exemples : `systeme-installe-20260917`, `avant-tryton-20260922`.

Un snapshot sans date ni objet est inutilisable trois semaines plus tard.

---

## 2. Adressage

### Plage réservée aux VM

| Adresse | Attribution | Date |
|---|---|---|
| 192.168.75.249 | **Réservée pour `vm-production`** (migration à venir) | 17/09/2026 |
| 192.168.75.250 | `vm-erp-poc` | 17/09/2026 |
| 192.168.75.251 | Libre | |
| 192.168.75.252 | Libre | |
| 192.168.75.253 | Libre | |

`192.168.75.254` est écartée : réservée par convention à un équipement réseau.

`192.168.75.249` est mise de côté dès maintenant pour la future VM de production.
Elle ne doit pas être attribuée à une autre machine, même temporairement : la
migration supposera une bascule où l'ancienne et la nouvelle configuration
coexistent brièvement.

**Ce tableau est mis à jour à chaque création ou suppression de VM.** C'est le seul
plan d'adressage existant pour ce sous-réseau ; sa tenue conditionne la fiabilité
de tout ce qui suit.

### Règle d'attribution

1. Adresse fixe uniquement. Aucune VM en DHCP : une adresse qui change casse les
   accès, les configurations et les sauvegardes.
2. Prendre la première adresse libre de la plage, dans l'ordre.
3. **Vérifier avant d'attribuer**, même si le tableau la déclare libre :
   ```bash
   sudo arping -c 3 -w 3 -I enp2s0f1 192.168.75.<n>
   ```
   Aucune réponse = adresse libre au moment du test.
4. Mettre à jour le tableau immédiatement, pas « plus tard ».

### Réserve connue
Les cinq adresses ont été vérifiées libres par ARP le 17/09/2026. Cela n'établit pas
qu'elles sont exclues d'un éventuel pool DHCP porté par la passerelle
`192.168.75.1`. Tant que cette plage n'est pas confirmée, un conflit reste
théoriquement possible. Point ouvert n°2 de l'état des lieux.

### Extension de la plage
Si les cinq adresses viennent à manquer, ne pas prendre une adresse au hasard
ailleurs. Obtenir d'abord la plage DHCP, puis réserver un intervalle contigu et
le documenter ici.

---

## 3. Réseau

### Mode retenu : pont
Les VM sont raccordées au pont `br0`, porté par l'interface physique `enp2s0f1`.
Elles obtiennent une adresse sur le réseau interne et sont joignables comme des
machines à part entière.

Le NAT de libvirt (`192.168.122.0/24`) n'est pas utilisé et reste désactivé.

### Sous-réseaux à ne pas utiliser
Les plages suivantes sont déjà occupées par des réseaux Docker de l'hôte. Aucune
VM, aucun réseau interne de VM ne doit les recouvrir :

```
172.17 à 172.25 (/16)          172.30.0.0/24
192.168.78, 79, 80, 82, 89, 90 (/24)
```

Plages disponibles : 172.26 à 172.29, 172.31, 192.168.81, 192.168.83 à 192.168.88,
192.168.91 et suivantes.

### Interfaces de secours
`eno1`, `eno2` et `enp2s0f0` ne sont pas câblées et portent des adresses de
l'ancien plan `192.168.78.0/24`, lequel est également utilisé par un réseau Docker.
**Ne pas les activer** sans avoir traité ce conflit.

---

## 4. Stockage

### Emplacement
`/var/lib/libvirt/images/`, sur le volume racine (1,3 To disponibles).

### Format
qcow2 exclusivement. Il permet les snapshots et n'occupe que l'espace réellement
écrit, ce qui évite d'immobiliser 200 Go dès la création d'une VM.

### Dimensionnement
Dimensionner au besoin réel, pas au maximum possible. Un disque qcow2 s'agrandit
sans réinstaller la VM ; le réduire est beaucoup plus délicat.

### Surveillance
L'espace disponible sur `/` est surveillé avant chaque création de VM :
```bash
df -h /var/lib/libvirt/images/
```
Le format qcow2 grandit à l'usage : une VM déclarée à 200 Go peut n'en occuper que
20 au départ et atteindre son maximum plus tard. Ne jamais provisionner une somme de
disques virtuels qui, une fois pleins, dépasserait l'espace réel.

---

## 5. Dimensionnement et réserve

### Réserve garantie à l'hôte
L'hôte conserve en permanence, pour lui-même et pour les applications de
production :

- **16 Go de mémoire minimum**
- **4 cœurs minimum**

Cette réserve n'est pas négociable. Elle protège les vingt conteneurs en service.

### Plafond alloué aux VM
Tant que les applications de production tournent directement sur l'hôte :

- somme des mémoires allouées aux VM : **32 Go maximum** (sur 62 Go)
- la mémoire est réellement consommée par une VM allumée, contrairement au disque

### Processeurs
Le dépassement du nombre de threads physiques est admis : les VM ne consomment pas
leurs vCPU en permanence. Sur 20 threads, allouer 8 vCPU à une VM est raisonnable.
Éviter néanmoins qu'une seule VM déclare plus de la moitié des threads.

### Table d'allocation

| VM | vCPU | Mémoire | Disque | Statut |
|---|---|---|---|---|
| `vm-erp-poc` | 8 | 16 Go | 200 Go | À créer |
| `vm-production` | à définir | à définir | à définir | Migration à cadrer |
| | | | | |
| **Total alloué** | **8** | **16 Go** | **200 Go** | |
| **Reste disponible** | | **16 Go sur le plafond** | | |

Ce tableau est mis à jour à chaque création, modification ou suppression de VM.

**Note sur le plafond.** La limite de 32 Go s'entend tant que les applications de
production tournent directement sur l'hôte. Le jour où elles migrent dans
`vm-production`, l'hôte n'a plus besoin que de sa réserve d'hyperviseur et le
plafond devra être relevé. À réviser à ce moment-là, pas avant.

### Ajustement
La mémoire et le nombre de vCPU d'une VM se modifient sans réinstallation, VM
arrêtée. Il vaut donc mieux démarrer bas et augmenter si nécessaire que l'inverse.

---

## 6. GPU

L'hôte dispose d'une carte NVIDIA RTX PRO 2000 Blackwell, **actuellement utilisée en
production** par le conteneur `ollama`, ainsi que par les applications métier qui
consomment les modèles locaux via `litellm`.

### Règle courante
**Aucune VM ne reçoit le GPU sans décision explicite et planifiée.**

Passer le GPU à une VM (PCI passthrough) suppose de le détacher de l'hôte, ce qui
rendrait la pile d'IA locale inopérante. L'opération nécessite en outre l'activation
de l'IOMMU dans le BIOS et une modification des paramètres de démarrage du noyau,
donc un redémarrage et une fenêtre de maintenance.

### Attribution cible
Le GPU ne peut appartenir qu'à **une seule machine à la fois**. Il est réservé à la
future `vm-production`, qui regroupera l'ensemble des applications actuellement
hébergées directement sur l'hôte.

Conséquence : le passage du GPU et la migration des conteneurs sont une seule et
même opération. On ne peut pas détacher le GPU de l'hôte avant que les applications
qui l'utilisent aient migré.

Aucune autre VM ne recevra de GPU tant que la machine n'en compte qu'un.

### À vérifier avant la migration
Le regroupement IOMMU de la carte : si elle partage son groupe avec d'autres
périphériques, ceux-ci devront être passés avec elle, ce qui n'est pas toujours
possible. Vérification en lecture seule, à faire en amont du chantier.

---

## 7. Snapshots

### Ce qu'ils sont
Un outil de travail permettant de revenir à un état antérieur en quelques secondes.
Ils vivent sur le même disque que la VM.

### Ce qu'ils ne sont pas
Une sauvegarde. Un disque défaillant emporte la VM et ses snapshots. Ils ne
remplacent aucune politique de sauvegarde.

### Règles d'usage
1. Un snapshot est pris **avant toute opération à risque** : installation lourde,
   montée de version, modification de configuration système.
2. Il est **supprimé une fois l'opération validée**. Les snapshots empilés dégradent
   les performances disque et compliquent la lecture de l'historique.
3. Un snapshot conservé plus d'une semaine doit avoir une raison documentée.
4. Nommage selon la section 1.

### Commandes usuelles
```bash
virsh snapshot-create-as <vm> <nom> --description "<raison>"
virsh snapshot-list <vm>
virsh snapshot-revert <vm> <nom>
virsh snapshot-delete <vm> <nom>
```

---

## 8. Sauvegarde

Chaque VM relève d'une politique de sauvegarde définie selon son usage, précisée
dans sa fiche.

**Pour `vm-erp-poc`** : la reproductibilité depuis le dépôt Git est le niveau
principal, conformément au plan projet du POC. Aucune donnée réelle n'y transite.

**Pour une VM hébergeant des données de production** : une politique de sauvegarde
distincte est obligatoire avant la mise en service. Une VM non sauvegardée ne
reçoit pas de données de production.

À documenter : l'agent Synology Active Backup présent sur l'hôte couvre-t-il
`/var/lib/libvirt/images/` ? Point ouvert n°5 de l'état des lieux.

---

## 9. Accès et administration

### Administration
Par SSH, authentification par clé uniquement. Connexion root désactivée,
authentification par mot de passe désactivée.

La console libvirt (`virsh console`) sert exclusivement de secours, quand le réseau
de la VM est inaccessible.

### Responsables
Administrateurs de l'hôte et des VM : le développeur SI et le tuteur.

### Secrets
Aucun mot de passe, clé privée ou identifiant dans le dépôt Git. Les fichiers de
configuration versionnés sont expurgés de tout secret.

---

## 10. Cycle de vie d'une VM

### Création
1. Vérifier la réserve disponible dans la table d'allocation (section 5).
2. Choisir et vérifier l'adresse IP (section 2).
3. Créer la VM selon les conventions de nommage et de stockage.
4. Installer le système : pas d'environnement graphique, serveur SSH uniquement.
5. Durcir : utilisateur non privilégié, clé SSH, root et mot de passe désactivés.
6. Prendre un snapshot `systeme-installe-<date>`.
7. Mettre à jour les tables d'adressage et d'allocation.
8. Rédiger la fiche de la VM (section 11).

### Mise hors service
1. Vérifier qu'aucune donnée utile ne reste sur la VM.
2. Arrêter la VM, attendre une période de grâce définie avant suppression.
3. Supprimer la VM et son disque.
4. Libérer l'adresse dans la table d'adressage, libérer les ressources dans la
   table d'allocation.
5. Consigner la suppression et sa date.

Une adresse libérée n'est pas réattribuée immédiatement : laisser passer quelques
semaines évite les confusions dans les journaux et les configurations résiduelles.

---

## 11. Documentation obligatoire

Chaque VM dispose d'une fiche dans `docs/infra/<nom-vm>.md`, contenant au minimum :

- usage et responsable ;
- configuration (vCPU, mémoire, disque, système) ;
- adresse IP et nom d'hôte ;
- date de création ;
- procédure de recréation ;
- politique de sauvegarde applicable ;
- particularités éventuelles.

Une VM sans fiche est une VM que personne ne saura reprendre.

---

## 12. Règles impératives sur l'hôte

Ces interdictions protègent les applications de production. Elles s'appliquent à
toute personne intervenant sur `debian2018`.

### Interdit
- `docker system prune` et toute commande de nettoyage global Docker, sous toutes
  ses formes, y compris `prune -a` et `volume prune`.
- Toute modification de `/etc/docker/daemon.json` ou redémarrage du démon Docker
  hors fenêtre convenue.
- Tout redémarrage de l'hôte hors fenêtre convenue.
- La suppression d'un réseau, volume ou conteneur dont l'usage n'est pas
  formellement établi.
- L'activation des interfaces `eno1`, `eno2` ou `enp2s0f0` sans traitement préalable
  du conflit `192.168.78.0/24`.
- Le détachement du GPU de l'hôte sans décision planifiée.

### Obligatoire
- Sauvegarder tout fichier de configuration système avant modification, avec
  horodatage.
- Consigner toute modification de la configuration de l'hôte dans le dépôt Git.
- Vérifier l'état des conteneurs après toute opération touchant l'hôte :
  ```bash
  docker ps --format '{{.Names}}\t{{.Status}}'
  ```

---

## 13. Historique des versions

| Version | Date | Évolution |
|---|---|---|
| V1.00 | 17/09/2026 | Création, avant mise en place de la première VM. |
| V1.01 | 17/09/2026 | Découpage cible acté : une seule VM de production recevant le GPU. Adresse `.249` réservée pour `vm-production`. Section 6 (GPU) et table d'allocation complétées. |
