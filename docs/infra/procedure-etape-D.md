# Infrastructure — Procédure de l'étape D : pont réseau et correctifs

**Version : V1.01**
**Date : 17/09/2026**
**Serveur : `debian2018` — 192.168.75.10**
**Plan de référence : `26-09-16_SI-Infra_Plan-virtualisation_V1.00.md`**

> **Évolution depuis la V1.00.** Une tentative d'application du pont à chaud, sans
> redémarrage, a été menée le 17/09 à 15 h. Elle a échoué : `ifupdown` ne permet pas
> de basculer une interface active vers un pont sans redémarrage. La procédure est
> revue en conséquence. Le fichier de configuration a été validé et est en place ;
> il sera appliqué au redémarrage.

---

## 0. Ce que fait cette procédure

Deux opérations dans la même fenêtre :

1. Transformer l'interface `enp2s0f1` en pont réseau `br0`, pour que les machines
   virtuelles obtiennent une adresse sur le réseau interne.
2. Appliquer les correctifs de sécurité en attente, dont le noyau, qui imposent un
   redémarrage.

L'adresse du serveur ne change pas : `192.168.75.10` passe de l'interface physique
au pont.

## Pourquoi c'est l'étape à risque

`enp2s0f1` est la seule interface câblée du serveur. Elle porte l'accès SSH et les
vingt conteneurs de production. Une erreur de configuration rend la machine
injoignable.

`eno1`, `eno2` et `enp2s0f0` ne sont pas des solutions de repli : elles ne sont pas
câblées, et la carte intégrée a provoqué des coupures réseau par le passé, ce qui a
motivé la bascule vers `enp2s0f1` le 30/06/2026. Vérifié le 17/09 : `enp2s0f1` est
sur la carte PCIe Intel I350-T2 (bus `02:`), physiquement distincte de la carte
intégrée (bus `01:`), et n'accumule aucune erreur.

---

## 1. Avant la fenêtre

- [x] **Fenêtre convenue** : redémarrage programmé le 18/09 à 12 h.
- [x] **Console physique testée** : écran et clavier branchés, invite obtenue,
      authentification réussie.
- [x] **Identifiants disponibles** : mot de passe de `administrateur` connu. En
      console, la clé SSH ne sert à rien.
- [x] **`docker-compose.yml`** de `/opt/ressource-preprod` corrigé (`restart:
      unless-stopped` sur `db`, `backend`, `frontend`). À appliquer pendant la
      fenêtre.
- [x] **Cause de la panne de `eno1`** : carte réseau intégrée, remplacée par la
      carte PCIe Intel I350-T2 en juin 2026. Vérifié : `enp2s0f1` est sur cette
      carte PCIe, sur un bus distinct, sans erreur accumulée.
- [x] **Fichier `/etc/network/interfaces`** modifié et vérifié le 17/09.
- [x] **Sauvegardes** dans `~/etape-D/` et `/root/interfaces.secours`.

### Matériel à avoir sur place
Écran, clavier, et le câble d'alimentation de l'écran. Un PowerEdge T630 dispose de
ports VGA et USB en façade et à l'arrière.

---

## 2. Étape par étape

**État au 17/09/2026 :** les sections 2.1 à 2.5 sont **déjà réalisées**. Le fichier
de configuration est en place et vérifié, les sauvegardes existent.

**Le 18/09 à 12 h, reprendre directement à la section 2.6.**

Durée estimée pour ce qui reste : 45 minutes à 1 heure.

### 2.1 Sauvegardes (fait le 17/09)

```bash
mkdir -p ~/etape-D && cd ~/etape-D
DATE=$(date +%F-%H%M)

sudo cp /etc/network/interfaces ~/etape-D/interfaces.bak-$DATE
ip -br addr show > interfaces-avant.txt
ip route > routes-avant.txt
docker ps --format '{{.Names}}\t{{.Status}}' | sort > conteneurs-avant.txt
sudo nft list ruleset > nftables-avant.txt

echo "Sauvegarde : interfaces.bak-$DATE"
ls -l ~/etape-D/
```

**Notez la valeur de `$DATE`**, elle servira au retour arrière.

### 2.2 Préparation du filet de sécurité (fait le 17/09)

Un script qui restaure automatiquement la configuration si l'accès est perdu.

```bash
sudo tee /usr/local/sbin/retour-reseau.sh > /dev/null <<'EOF'
#!/bin/bash
# Restaure la configuration réseau si le pont n'a pas été validé.
if [ -f /run/pont-valide ]; then
    logger "retour-reseau : pont valide, aucune action"
    exit 0
fi
logger "retour-reseau : pont NON valide, restauration"
cp /root/interfaces.secours /etc/network/interfaces
systemctl restart networking
EOF

sudo chmod +x /usr/local/sbin/retour-reseau.sh
sudo cp /etc/network/interfaces /root/interfaces.secours
```

Vérifier que le fichier de secours est bien l'ancienne configuration :

```bash
sudo grep -A3 "enp2s0f1" /root/interfaces.secours
```

### 2.3 Nouvelle configuration réseau (fait le 17/09)

```bash
sudo nano /etc/network/interfaces
```

**Remplacer la strophe de `enp2s0f1`** par les deux blocs suivants. Ne pas toucher
aux strophes `lo`, `eno1`, `eno2` et `enp2s0f0`.

Ancien bloc, à supprimer ou commenter :

```
auto enp2s0f1
iface enp2s0f1 inet static
    address 192.168.75.10
    netmask 255.255.255.0
    gateway 192.168.75.1
    dns-nameservers 9.9.9.9 1.1.1.1
```

Nouveau contenu :

```
# Interface principale — devenue port du pont br0 le <date>
# Carte Intel I350-T2 (PCIe), port 2 — câble actif
auto enp2s0f1
iface enp2s0f1 inet manual

# Pont pour les machines virtuelles — porte l'adresse du serveur
auto br0
iface br0 inet static
    address 192.168.75.10
    netmask 255.255.255.0
    gateway 192.168.75.1
    dns-nameservers 9.9.9.9 1.1.1.1
    bridge_ports enp2s0f1
    bridge_stp off
    bridge_fd 0
    bridge_maxwait 0
```

**Points de vigilance**
- `bridge_ports enp2s0f1` : seule cette interface entre dans le pont. Les trois
  autres restent dehors, pour ne pas réveiller le conflit `192.168.78.0/24`.
- `bridge_stp off` : le protocole d'arbre couvrant est inutile avec un seul port et
  retarde la montée de l'interface.
- L'adresse ne change pas, la passerelle et les DNS non plus.

Vérifier la cohérence du fichier avant d'appliquer :

```bash
grep -n -E "auto|iface|bridge_" /etc/network/interfaces
```

### 2.4 Pourquoi l'application à chaud ne fonctionne pas

**Constat du 17/09/2026, à conserver.** La V1.00 de cette procédure prévoyait
d'appliquer le pont sans redémarrage, avec un retour automatique en cas de perte
d'accès. La tentative a échoué.

Commande lancée :
```bash
sudo bash -c 'ifdown enp2s0f1 ; ifup br0'
```

Résultat :
```
ifdown: interface enp2s0f1 not configured
RTNETLINK answers: File exists
ifup: failed to bring up br0
```

Deux causes :

1. `ifdown` considère `enp2s0f1` comme non gérée, parce que le fichier de
   configuration a changé depuis le démarrage et ne correspond plus à l'état
   enregistré par `ifupdown`.
2. L'adresse `192.168.75.10` étant toujours portée par `enp2s0f1`, le pont n'a pas
   pu la prendre. Il a été créé mais est resté dans un état incomplet : l'adresse
   figurait simultanément sur les deux interfaces.

Nettoyage effectué, sans incident et sans coupure de la session SSH :
```bash
sudo ip link set br0 down
sudo ip link delete br0 type bridge
```

**Conclusion.** Avec `ifupdown`, la bascule d'une interface active vers un pont se
fait au redémarrage. Ce n'est pas un défaut de la configuration : le fichier était
correct et le reste. La procédure passe donc directement aux correctifs puis au
redémarrage.

**Le filet de sécurité de la section 2.2 ne protège pas d'un redémarrage raté.** Il
se déclenche après une bascule à chaud, pas après un `reboot`. La protection réelle
est la console physique.

### 2.5 Vérification du fichier avant redémarrage

```bash
grep -n -E "^auto|^iface|address|bridge_" /etc/network/interfaces
```

Contrôler trois points :
- une seule occurrence de `192.168.75.10`, portée par `br0` ;
- `iface enp2s0f1 inet manual`, sans adresse ;
- les strophes `lo`, `eno1`, `eno2` et `enp2s0f0` intactes, hors du pont.

Vérifié conforme le 17/09/2026.

Vérifier aussi qu'aucune configuration du système ne référence le nom de
l'interface, qui ne portera plus l'adresse :

```bash
sudo grep -rn "enp2s0f1" /etc --exclude-dir=network 2>/dev/null
sudo grep -rn "enp2s0f1" /opt /usr/local 2>/dev/null
```

Aucun résultat le 17/09/2026 : rien ne dépend du nom de l'interface.

### 2.6 Vérification des conteneurs avant redémarrage

```bash
docker ps --format '{{.Names}}\t{{.Status}}' | sort > ~/etape-D/conteneurs-avant.txt
wc -l ~/etape-D/conteneurs-avant.txt
docker ps -a --format '{{.Names}}\t{{.HostConfig.RestartPolicy.Name}}' 2>/dev/null | grep -v unless-stopped | grep -v always
```

La dernière commande doit ne rien renvoyer : tout conteneur sans politique de
redémarrage ne remonterait pas. Quatre cas ont été corrigés le 17/09
(`ressource-backend-1`, `ressource-db-1`, `ressource-frontend-1`, `pedantic_bassi`).

### 2.7 Correction du Compose de `ressource-preprod` (5 min)

```bash
cd /opt/ressource-preprod
sudo docker compose config --quiet && echo "syntaxe OK"
sudo docker compose up -d
docker inspect ressource-backend-1 --format '{{.HostConfig.RestartPolicy.Name}}'
```

Attendu : `unless-stopped`.

### 2.8 Correctifs de sécurité (10 min)

```bash
sudo apt-get update
sudo apt-get upgrade -y
```

Paquets concernés : noyau 6.12.90 vers 6.12.94, `linux-libc-dev`, `libssh2-1t64`,
`python3-urllib3`, Docker 29.5.3 vers 29.6.1, `containerd.io`,
`docker-compose-plugin`.

**La mise à jour de Docker redémarre le démon** : tous les conteneurs s'arrêtent et
redémarrent. C'est attendu, et c'est pourquoi l'opération se fait dans cette
fenêtre.

Les erreurs 404 sur les dépôts `bookworm` sont sans gravité, ce sont des résidus de
la montée de version vers Debian 13.

### 2.9 Redémarrage (10 min)

```bash
sudo touch /run/pont-valide.permanent
sync
sudo reboot
```

**Rester devant la console pendant le redémarrage.** L'initramfs a été régénéré lors
de l'installation de libvirt à l'étape C : c'est le premier démarrage qui l'utilise.

### 2.10 Vérifications après redémarrage (15 min)

**Le pont est monté à ce moment précis.** C'est le vrai test de la configuration.

Dans l'ordre :

```bash
# 1. Accès réseau — si cette étape échoue, passer au retour arrière (section 3)
ssh administrateur@192.168.75.10

# 2. Le pont est-il en place ?
ip -br addr show | grep -E 'br0|enp2s0f1'
ip route | head -2
bridge link show
```

Résultat attendu : `br0` porte `192.168.75.10/24`, `enp2s0f1` est UP **sans
adresse**, la route par défaut passe par `br0`, et `bridge link show` montre
`enp2s0f1` rattachée au pont.

```bash
# 3. Noyau
uname -r

# 4. Conteneurs — le point le plus important
docker ps --format '{{.Names}}\t{{.Status}}' | sort > ~/etape-D/conteneurs-apres.txt
diff ~/etape-D/conteneurs-avant.txt ~/etape-D/conteneurs-apres.txt

# 5. Aucun conteneur en erreur ou en boucle
docker ps -a --filter "status=restarting" --filter "status=exited" \
  --format '{{.Names}}\t{{.Status}}'

# 6. Connectivité sortante
ping -c3 192.168.75.1
ping -c3 deb.debian.org

# 7. libvirt
virsh list --all

# 8. GPU et pile IA
nvidia-smi | head -12
docker logs --tail 20 ollama
```

**Test applicatif depuis un poste client**, sur plusieurs applications à travers le
reverse proxy. Le `diff` prouve que les conteneurs tournent, pas qu'ils sont
joignables.

Si un conteneur manque, le relancer par son projet Compose plutôt que par
`docker start`, pour rester cohérent avec sa configuration.

---

## 3. Retour arrière

### Si le réseau ne remonte pas après redémarrage

C'est le scénario à couvrir en priorité. Depuis la **console physique** (écran et
clavier branchés sur le serveur) :

```bash
sudo cp /root/interfaces.secours /etc/network/interfaces
sudo systemctl restart networking
ip -br addr show
ip route | head -2
```

`/root/interfaces.secours` contient l'ancienne configuration, vérifiée identique à
la sauvegarde horodatée `~/etape-D/interfaces.bak-2026-09-17-1458`.

Puis vérifier l'accès SSH depuis un poste client, et l'état des conteneurs.

### Si le serveur ne démarre pas

Au menu de démarrage, choisir « Advanced options » et sélectionner un noyau
antérieur : `6.12.69+deb13-amd64` ou `6.12.63+deb13-amd64`. Les trois noyaux sont
installés.

### Si un conteneur ne remonte pas

```bash
docker inspect <conteneur> --format '{{index .Config.Labels "com.docker.compose.project.working_dir"}}'
cd <repertoire> && sudo docker compose up -d
docker logs --tail 50 <conteneur>
```

### Note sur le filet de sécurité automatique

Le script `/usr/local/sbin/retour-reseau.sh` et le minuteur `systemd-run` décrits en
section 2.2 ne protègent **que** d'une bascule à chaud ratée. Ils ne se déclenchent
pas après un redémarrage.

La protection réelle pour cette étape est la console physique. Le fichier
`/run/pont-valide` est de toute façon effacé au redémarrage, `/run` étant en mémoire.

---

## 4. Compte rendu à remplir

| Élément | Valeur |
|---|---|
| Date et heure de début | |
| Date et heure de fin | |
| Durée d'indisponibilité réelle | |
| Personnes présentes | |
| Pont créé | oui / non |
| Correctifs appliqués | oui / non |
| Noyau après redémarrage | |
| Conteneurs manquants au redémarrage | |
| Incidents rencontrés | |
| Retour arrière déclenché | oui / non |

### À faire après la fenêtre

- [ ] Compléter la section 2 de `docs/infra/installation-hote.md`.
- [ ] Verser `/etc/network/interfaces` (version avec pont) dans le dépôt Git.
- [ ] Consigner la décision dans `docs/decisions.md`.
- [ ] Mettre à jour le tableau de suivi du plan de virtualisation.
- [ ] Supprimer `/usr/local/sbin/retour-reseau.sh` et `/root/interfaces.secours`
      une fois la configuration stabilisée, ou les conserver en les documentant.

---

## 5. Ce qui vient ensuite

Étape E : création de la VM `vm-erp-poc` sur `192.168.75.250`, raccordée au pont
`br0`. Sans risque pour la production, et sans nouvelle fenêtre.

---

## 6. Historique des versions

| Version | Date | Évolution |
|---|---|---|
| V1.00 | 17/09/2026 | Création. |
| V1.01 | 17/09/2026 | Échec de l'application à chaud documenté (section 2.4). La bascule se fait désormais au redémarrage. Retour arrière revu : le filet automatique ne couvre pas un redémarrage. Vérifications post-redémarrage renforcées sur l'état du pont. |
