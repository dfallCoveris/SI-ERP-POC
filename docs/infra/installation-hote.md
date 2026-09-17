# Infrastructure — Installation de l'hôte de virtualisation

**Serveur : `debian2018` — 192.168.75.10**
**Plan de référence : `26-09-16_SI-Infra_Plan-virtualisation_V1.00.md`**

Ce document est complété au fil des étapes. Il doit permettre de reconstruire
l'hôte à l'identique.

---

## 1. Hyperviseur (étape C)

**Date : 17/09/2026**
**Statut : terminé**

### Objectif
Installer KVM et libvirt sans toucher au réseau, sans redémarrer Docker, sans
redémarrer la machine.

### État avant intervention

Relevé sauvegardé dans `~/etat-avant-kvm/` sur l'hôte :

| Fichier | Contenu |
|---|---|
| `paquets.txt` | `dpkg -l` complet (520 lignes) |
| `conteneurs.txt` | Conteneurs et statuts (44 lignes) |
| `nftables.txt` | Règles nftables (275 lignes) |
| `iptables.txt` | Règles iptables (107 lignes) |
| `interfaces.txt` | `ip -br addr show` (65 lignes) |
| `routes.txt` | Table de routage (21 lignes) |

Ce relevé sert de référence de comparaison et de base à un éventuel retour arrière.

### Prérequis vérifiés (étape A)

| Vérification | Résultat |
|---|---|
| Drapeau `vmx` (VT-x) | Présent sur les 20 threads |
| Modules `kvm` et `kvm_intel` | Déjà chargés avant installation |
| `/dev/kvm` | Présent |

Aucun module à charger : le matériel était déjà prêt.

### Commande d'installation

```bash
sudo apt-get update
sudo apt-get install -y qemu-system-x86 libvirt-daemon-system libvirt-clients \
                        virtinst bridge-utils
```

Volume : 360 paquets installés, 3 mis à jour, 262 Mo téléchargés, 978 Mo occupés.
Le nombre élevé s'explique par les dépendances multimédia et graphiques de QEMU,
sans incidence sur un usage serveur.

### Versions installées

| Composant | Version |
|---|---|
| libvirt | 11.3.0-3+deb13u3 |
| QEMU (système x86) | 1:10.0.13+ds-0+deb13u1 |
| virtinst / virt-install | 1:5.0.0-5+deb13u1 |
| bridge-utils | 1.7.1-4+b1 |
| python3-libvirt | 11.3.0-1 |

### Services activés

- `libvirtd` (activation par socket)
- `virtlogd`
- `virtlockd`
- `libvirt-guests`
- `open-iscsi` et `iscsid` (dépendances du pilote de stockage libvirt)

**Note sur `libvirtd`** : `systemctl is-active libvirtd` renvoie `inactive` au repos.
Ce n'est pas une anomalie. Le service fonctionne en activation par socket : il
démarre à la première commande `virsh` et s'arrête ensuite. Le critère de bon
fonctionnement est la réponse de `virsh`, pas l'état permanent du service.

### Réseau NAT par défaut

Le plan prévoyait de désactiver le réseau `default` de libvirt
(`192.168.122.0/24`), inutile puisque le mode retenu est le pont.

**Aucune action n'a été nécessaire** : Debian 13 livre ce réseau déjà inactif.

```
 Nom       État      Démarrage automatique   Persistant
 default   inactif   non                     oui
```

Aucune interface `virbr0` n'a été créée. La définition du réseau reste présente
mais inerte ; elle est conservée en l'état.

### Vérifications après installation

| Contrôle | Commande | Résultat |
|---|---|---|
| Conteneurs Docker | `diff conteneurs.txt conteneurs-apres.txt` | Identiques, seules les durées d'exécution diffèrent |
| Réponse de libvirt | `virsh list --all` | Liste vide, sans erreur |
| Vue matérielle | `virsh nodeinfo` | 20 CPU, 10 cœurs, 2 threads, 65 750 728 KiB |
| Interfaces réseau | `ip -br addr show` | `enp2s0f1` inchangée, pas de `virbr0` |
| Réseaux libvirt | `virsh net-list --all` | `default` inactif |

Aucun conteneur perdu, aucun redémarrage de service applicatif, aucune modification
de la configuration réseau.

### Droits d'administration

L'utilisateur `administrateur` a été ajouté au groupe `libvirt` :

```bash
sudo usermod -aG libvirt administrateur
```

`virsh` fonctionne sans `sudo`. Le changement de groupe est effectif à la prochaine
ouverture de session.

### Retour arrière

Aucune donnée en jeu à ce stade. En cas de besoin :

```bash
sudo systemctl stop libvirtd
sudo apt-get remove --purge -y libvirt-daemon-system libvirt-clients virtinst
sudo apt-get autoremove -y
```

### Points relevés pendant l'installation

**1. Régénération de l'initramfs.** L'installation d'`open-iscsi` a déclenché la
reconstruction de `/boot/initrd.img-6.12.90+deb13.1-amd64`. Sans effet tant que la
machine ne redémarre pas, mais le prochain démarrage (étape D) utilisera cet
initramfs régénéré. À garder en tête lors de la vérification post-redémarrage.

**2. Dépôts APT obsolètes.** Deux entrées pointant vers Debian 12 renvoient une
erreur 404 à chaque `apt update` :

```
http://archive.debian.org/debian bookworm Release
http://archive.debian.org/debian-security bookworm-security Release
```

Résidus de la montée de version vers Debian 13. Sans gravité : les dépôts trixie
fonctionnent normalement. À nettoyer hors du cadre de ce projet.

**3. Hooks Synology.** L'installation déclenche `synosnap` avant et après
l'opération, confirmant que l'agent Active Backup surveille les modifications de
paquets.

### Critère de fin

Atteint. `virsh list --all` répond et les applications existantes sont inchangées.

---

## 2. Pont réseau (étape D)

**Statut : à faire.**

À compléter pendant la fenêtre de maintenance.

### Prérequis non satisfaits

- Accès iDRAC : adresse, identifiants et connexion de test à obtenir. **Bloquant.**
- Fenêtre de maintenance à convenir.

### À consigner

- configuration réseau avant et après, horodatée ;
- correctifs de sécurité appliqués ;
- durée de l'interruption ;
- état des conteneurs après redémarrage ;
- incidents éventuels.
