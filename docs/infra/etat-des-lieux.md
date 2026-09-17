# Infrastructure — État des lieux de l'hôte de virtualisation

**Date du relevé : 17/09/2026**
**Serveur : `debian2018` — 192.168.75.10**
**Étape A du plan `26-09-16_SI-Infra_Plan-virtualisation_V1.00.md`**

Tous les relevés ont été effectués en lecture seule, à l'exception de trois
modifications signalées en section 8.

---

## 1. Identification de la machine

| Élément | Valeur |
|---|---|
| Constructeur | Dell Inc. |
| Modèle | PowerEdge T630 |
| Numéro de série | HWKPCM2 |
| Système | Debian GNU/Linux 13 (trixie) |
| Noyau au moment du relevé | 6.12.90+deb13.1-amd64 |
| Nom d'hôte | debian2018 |

---

## 2. Ressources

| Élément | Valeur |
|---|---|
| Processeur | Intel Xeon E5-2630 v4 à 2,20 GHz (3,10 GHz max) |
| Cœurs / threads | 10 cœurs, 20 threads, 1 socket |
| Mémoire totale | 62 Go |
| Mémoire disponible au relevé | 49 Go |
| Swap | 63 Go |
| Racine `/` | 1,6 To, 236 Go utilisés, **1,3 To disponibles** |
| `/boot` | 944 Mo |

Ces ressources couvrent largement le besoin du POC et laissent de la marge pour
les VM suivantes.

### Noyaux installés
- 6.12.63+deb13-amd64 (signalé comme obsolète par APT)
- 6.12.69+deb13-amd64
- 6.12.90+deb13.1-amd64 (actif)

La présence de plusieurs noyaux permet de redémarrer sur une version antérieure si
un correctif pose problème. À conserver jusqu'à validation de l'étape D.

---

## 3. Support de la virtualisation

| Vérification | Résultat |
|---|---|
| Drapeau `vmx` (VT-x) | Présent sur les 20 threads |
| Module `kvm_intel` | Chargé |
| Module `kvm` | Chargé |
| `/dev/kvm` | Présent, groupe `kvm` |

**Conclusion** : l'étape C (installation de KVM et libvirt) ne rencontrera aucun
obstacle matériel. Les modules sont déjà actifs.

---

## 4. Matériel graphique

| Adresse PCI | Carte | Usage |
|---|---|---|
| 04:00.0 | NVIDIA RTX PRO 2000 Blackwell (16 Go) | **En service** |
| 09:00.0 | Matrox G200eR2 | Contrôleur d'administration (iDRAC) |

### GPU NVIDIA — en production

| Élément | Valeur |
|---|---|
| Pilote | 580.76.05 |
| CUDA | 13.0 |
| Mémoire occupée au relevé | 11 024 Mo sur 16 311 |
| Processus | `/usr/local/bin/python3.12` (PID 2689347) |
| Conteneur déclarant le GPU | `ollama` (`DeviceRequests` : driver nvidia, 1 GPU) |

Le serveur héberge une pile d'IA locale (`ollama`, `open-webui`, `litellm`,
`searxng`) exploitant cette carte.

**Conséquence pour la migration future de la production.** Une VM devant héberger
`ollama` nécessitera un passage direct du GPU (PCI passthrough) : activation de
l'IOMMU dans le BIOS, modification des paramètres de démarrage du noyau, détachement
de la carte de l'hôte. C'est l'opération la plus délicate du chantier de migration.
À traiter dans le document de cadrage correspondant.

**Aucun impact sur la VM du POC**, qui n'a aucun besoin de GPU.

---

## 5. Réseau

### Interfaces physiques

| Interface | État | Adresse | Commentaire |
|---|---|---|---|
| `enp2s0f1` | **UP** | 192.168.75.10/24 | Interface principale, carte Intel I350-T2 port 2. Porte toute la production. |
| `enp2s0f0` | DOWN | 192.168.78.100/24 | Carte Intel I350-T2 port 1, pas de câble |
| `eno1` | DOWN | 192.168.78.101/24 | Carte intégrée port 1, pas de câble |
| `eno2` | DOWN | 192.168.78.102/24 | Carte intégrée port 2, ex-active, basculée le 30/06/2026 |

Les trois interfaces en DOWN portent des adresses de l'ancien plan d'adressage
(`192.168.78.0/24`), devenu obsolète. Aucun câble n'y est branché : vérifié par
`ethtool`, `Link detected: no` sur les trois.

### Configuration réseau

| Élément | Valeur |
|---|---|
| Mécanisme | `ifupdown` (service `networking` activé) |
| Fichier | `/etc/network/interfaces` |
| `/etc/network/interfaces.d/` | Vide |
| `systemd-networkd` | Désactivé |
| NetworkManager | Absent |
| Passerelle | 192.168.75.1 |
| DNS | 1.1.1.1 et 9.9.9.9 |

**C'est la configuration la plus simple pour créer un pont** : un seul fichier à
modifier, pas de mécanisme concurrent. Le fichier est déjà commenté et documente
l'historique des interfaces.

### Point de vigilance
Le sous-réseau `192.168.78.0/24` est à la fois configuré sur les trois interfaces
de secours et utilisé par le réseau Docker `siverbarres_optim_net`. Conflit latent,
sans effet tant que ces interfaces restent éteintes. Ne pas les activer sans avoir
traité ce point.

---

## 6. Plan d'adressage

### Méthode
Balayage ICMP des 254 adresses de `192.168.75.0/24`, puis vérification ARP ciblée
(`arping`) sur les candidates. L'ARP est plus fiable : une machine qui ignore le
ping répond tout de même à une requête ARP.

### Adresses occupées au 17/09/2026 (67)

```
1, 2, 3, 4, 6, 7, 8, 9, 10, 19, 20, 22, 25, 27, 29, 30, 31, 35, 38, 39,
40, 43, 44, 45, 47, 50, 52, 55, 56, 63, 65, 71, 87, 88, 90, 93, 101, 102,
131, 139, 141, 146, 151, 159, 162, 163, 167, 172, 173, 174, 176, 180, 181,
184, 187, 197, 211, 213, 214, 215, 219, 220, 223, 226, 231, 241, 248
```

Les adresses occupées sont dispersées sur toute la plage, ce qui suggère un serveur
DHCP distribuant sur un large intervalle, probablement porté par la passerelle.

### Plage réservée aux machines virtuelles

| Adresse | Attribution |
|---|---|
| 192.168.75.249 | Libre, réservée VM |
| **192.168.75.250** | **`vm-erp-poc`** |
| 192.168.75.251 | Libre, réservée VM |
| 192.168.75.252 | Libre, réservée VM |
| 192.168.75.253 | Libre, réservée VM |

`192.168.75.254` est volontairement écartée : souvent réservée par convention à un
équipement réseau.

Les cinq adresses ont été vérifiées libres par `arping` le 17/09/2026.

### Réserve importante
Cette vérification établit qu'aucune machine n'utilise ces adresses **au moment du
relevé**. Elle n'établit pas qu'elles sont exclues d'un éventuel pool DHCP. La
confirmation de la plage de distribution sur la passerelle `192.168.75.1` reste à
obtenir. Risque résiduel : faible, mais un conflit d'adresse resterait possible.

Ce plan d'adressage est mis à jour à chaque création de VM.

---

## 7. Sous-réseaux Docker existants

| Réseau | Sous-réseau |
|---|---|
| `bridge` (défaut) | 172.17.0.0/16 |
| `debian2018_network` | 172.18.0.0/16 |
| `pointage_default` | 172.19.0.0/16 |
| `app-revision_default` | 172.20.0.0/16 |
| `app-contact-qr-code_default` | 172.21.0.0/16 |
| `extract-api_default` | 172.22.0.0/16 |
| `kalitics-extract_default` | 172.23.0.0/16 |
| `speakr_default` | 172.24.0.0/16 |
| `ressource_default` | 172.25.0.0/16 |
| `kalitics_extract_production` | 172.30.0.0/24 |
| `siverbarres_optim_net` | 192.168.78.0/24 |
| `sivertoles_optim-net` | 192.168.79.0/24 |
| `suivi-heures_suivi-heures-net` | 192.168.80.0/24 |
| `metre-preprod_metre-preprod-net` | 192.168.82.0/24 |
| `sivertoles-beta_optim-net-beta` | 192.168.89.0/24 |
| `pointage-beta_pointage-beta-net` | 192.168.90.0/24 |

### Plages disponibles pour de futurs réseaux
172.26 à 172.29, 172.31, 192.168.81, 192.168.83 à 192.168.88, 192.168.91 et suivantes.

Le réseau NAT créé par défaut par libvirt (`192.168.122.0/24`) n'entre en conflit
avec aucun de ces réseaux.

---

## 8. Modifications effectuées pendant l'étape A

Trois écarts au principe de lecture seule, tous volontaires et documentés.

### 8.1 Politiques de redémarrage corrigées

Quatre conteneurs n'avaient aucune politique de redémarrage et ne seraient pas
remontés après le redémarrage prévu à l'étape D :

- `ressource-backend-1`
- `ressource-db-1`
- `ressource-frontend-1`
- `pedantic_bassi`

Correction appliquée par `docker update --restart unless-stopped`.

**Correction durable restant à faire** : le fichier
`/opt/ressource-preprod/docker-compose.yml` ne contient aucune directive `restart`
pour ses trois services (`db` ligne 14, `backend` ligne 33, `frontend` ligne 108).
Le `docker update` sera perdu au prochain `docker compose up`. À corriger pendant
la fenêtre de l'étape D, quand l'interruption est de toute façon prévue.

### 8.2 Conteneur `pedantic_bassi`

Identifié comme l'image `hello-world`, créée le 16/02/2026. Conteneur de test sans
usage. À supprimer (`docker rm -f pedantic_bassi`).

### 8.3 Installation de `arping`

Paquets `arping` et `libnet1` installés pour la vérification d'adresses. Sans
incidence sur les services existants.

---

## 9. Correctifs en attente

| Paquet | Version installée | Version disponible |
|---|---|---|
| `linux-image-amd64` | 6.12.90-2 | 6.12.94-1 (sécurité) |
| `linux-libc-dev` | 6.12.90-2 | 6.12.94-1 (sécurité) |
| `libssh2-1t64` | 1.11.1-1 | 1.11.1-1+deb13u1 (sécurité) |
| `python3-urllib3` | 2.3.0-3+deb13u1 | 2.3.0-3+deb13u2 (sécurité) |
| `docker-ce` et composants | 29.5.3 | 29.6.1 |
| `containerd.io` | 2.2.4 | 2.2.5 |
| `docker-compose-plugin` | 5.1.4 | 5.2.0 |

Le noyau impose un redémarrage. La mise à jour de Docker entraîne un redémarrage du
démon, donc l'arrêt momentané de tous les conteneurs.

Ces correctifs sont à appliquer pendant la fenêtre de l'étape D.

---

## 10. Accès de secours

La machine est un Dell PowerEdge T630 : elle dispose d'une carte d'administration
à distance **iDRAC**, ce que confirme la présence du contrôleur graphique Matrox
G200eR2. L'iDRAC donne accès à la console même système planté ou réseau principal
tombé, sur sa propre adresse IP indépendante.

`ipmitool` n'est pas installé sur le serveur, mais ce n'est pas nécessaire :
l'iDRAC s'utilise depuis un navigateur.

**À obtenir avant la fenêtre de l'étape D** :
1. l'adresse IP de l'iDRAC ;
2. les identifiants d'accès ;
3. une connexion de test effective.

Un accès de secours jamais testé ne constitue pas un accès de secours.

---

## 11. Sauvegarde de l'hôte

L'installation de paquets déclenche des hooks `synosnap` : un agent Synology Active
Backup for Business est actif sur la machine et sauvegarde le système.

À documenter : fréquence, rétention, et procédure de restauration. Utile à
connaître avant toute opération lourde.

---

## 12. Points restant ouverts

| # | Point | Criticité |
|---|---|---|
| 1 | Adresse IP et identifiants de l'iDRAC, connexion testée | **Bloquant pour l'étape D** |
| 2 | Plage de distribution du serveur DHCP sur `192.168.75.1` | Important |
| 3 | Date de la fenêtre de maintenance | À convenir |
| 4 | Correction durable du `docker-compose.yml` de `ressource-preprod` | À faire pendant la fenêtre |
| 5 | Politique de sauvegarde Synology (fréquence, rétention, restauration) | Documentaire |
| 6 | Conflit latent `192.168.78.0/24` entre interfaces de secours et réseau Docker | À traiter si ces interfaces sont un jour activées |

---

## 13. Conclusion de l'étape A

Le serveur est apte à devenir hôte de virtualisation : virtualisation matérielle
active, ressources largement suffisantes, configuration réseau simple à faire
évoluer, adresse IP identifiée et vérifiée.

L'étape B (conventions) peut être rédigée. L'étape C (installation de KVM et
libvirt) peut être menée sans attendre, puisqu'elle ne nécessite ni redémarrage ni
modification du réseau.

L'étape D reste conditionnée à l'obtention des accès iDRAC.
