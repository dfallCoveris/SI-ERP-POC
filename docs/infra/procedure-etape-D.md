# Infrastructure — Procédure de l'étape D : pont réseau et correctifs

**Version : V1.00**
**Date : 17/09/2026**
**Serveur : `debian2018` — 192.168.75.10**
**Plan de référence : `26-09-16_SI-Infra_Plan-virtualisation_V1.00.md`**

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
motivé la bascule vers `enp2s0f1` le 30/06/2026.

---

## 1. Avant la fenêtre

À faire les jours précédents, pas le jour même.

- [ ] **Fenêtre convenue** avec le tuteur, utilisateurs prévenus.
- [ ] **Console physique testée** : écran et clavier branchés sur le serveur,
      invite de connexion obtenue, authentification réussie. Un accès de secours
      jamais testé n'est pas un accès de secours.
- [ ] **Identifiants disponibles** : mot de passe de `administrateur` connu, et
      mot de passe root si défini. En console, la clé SSH ne sert à rien.
- [ ] **Correction du `docker-compose.yml`** de `/opt/ressource-preprod` : ajouter
      `restart: unless-stopped` aux services `db`, `backend` et `frontend`.
      Le fichier peut être modifié à l'avance, l'application se fera pendant la
      fenêtre.
- [ ] **Cause de la panne de `eno1`** identifiée auprès du tuteur : câble, port de
      switch, ou carte intégrée défaillante. Si la carte PCIe Intel I350-T2 est en
      cause, l'opération devient plus risquée et doit être reconsidérée.

### Matériel à avoir sur place
Écran, clavier, et le câble d'alimentation de l'écran. Un PowerEdge T630 dispose de
ports VGA et USB en façade et à l'arrière.

---

## 2. Étape par étape

Durée estimée : 45 minutes à 1 h 30 selon les vérifications.

### 2.1 Sauvegardes (5 min)

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

### 2.2 Préparation du filet de sécurité (5 min)

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

### 2.3 Nouvelle configuration réseau (10 min)

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

### 2.4 Application sans redémarrage (10 min)

**C'est le moment sensible.** La session SSH va être coupée quelques secondes.

Armer le retour automatique à 3 minutes :

```bash
sudo rm -f /run/pont-valide
sudo systemd-run --on-active=180 --unit=retour-reseau \
     /usr/local/sbin/retour-reseau.sh
```

Appliquer, en une seule commande pour que la coupure soit la plus courte possible :

```bash
sudo bash -c 'ifdown enp2s0f1 ; ifup br0'
```

La session SSH tombe. **Reconnectez-vous immédiatement** depuis votre poste :

```bash
ssh administrateur@192.168.75.10
```

Si la reconnexion réussit, valider dans les 3 minutes :

```bash
sudo touch /run/pont-valide
sudo systemctl stop retour-reseau.timer 2>/dev/null
echo "Pont valide."
```

**Si la reconnexion échoue** : ne rien faire, attendre. Le script restaure la
configuration précédente au bout de 3 minutes et le serveur redevient joignable.
Analysez alors le fichier avant de recommencer.

### 2.5 Vérification du pont (5 min)

```bash
ip -br addr show | grep -E 'br0|enp2s0f1'
ip route
bridge link show
```

Résultat attendu : `br0` porte `192.168.75.10/24`, `enp2s0f1` est UP sans adresse,
la route par défaut passe par `br0`.

```bash
ping -c3 192.168.75.1
ping -c3 deb.debian.org
```

### 2.6 Vérification des conteneurs (10 min)

```bash
docker ps --format '{{.Names}}\t{{.Status}}' | sort > ~/etape-D/conteneurs-pont.txt
diff ~/etape-D/conteneurs-avant.txt ~/etape-D/conteneurs-pont.txt
```

Seules les durées d'exécution doivent différer.

**Test applicatif réel** : depuis un poste client, ouvrir une application à travers
le reverse proxy. Le `diff` prouve que les conteneurs tournent, pas qu'ils sont
joignables.

À ce stade, si tout est conforme, le pont est en service. **La suite peut être
reportée si nécessaire.**

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

Dans l'ordre :

```bash
# 1. Accès réseau
ssh administrateur@192.168.75.10

# 2. Noyau et pont
uname -r
ip -br addr show | grep -E 'br0|enp2s0f1'
ip route

# 3. Conteneurs — le point le plus important
docker ps --format '{{.Names}}\t{{.Status}}' | sort > ~/etape-D/conteneurs-apres.txt
diff ~/etape-D/conteneurs-avant.txt ~/etape-D/conteneurs-apres.txt

# 4. Aucun conteneur en erreur ou en boucle
docker ps -a --filter "status=restarting" --filter "status=exited" \
  --format '{{.Names}}\t{{.Status}}'

# 5. libvirt
virsh list --all

# 6. GPU et pile IA
nvidia-smi | head -12
docker logs --tail 20 ollama
```

**Test applicatif depuis un poste client**, sur plusieurs applications.

Si un conteneur manque, le relancer par son projet Compose plutôt que par
`docker start`, pour rester cohérent avec sa configuration.

---

## 3. Retour arrière

### Si le réseau ne remonte pas après application du pont
Ne rien faire pendant 3 minutes : le script `retour-reseau.sh` restaure la
configuration précédente automatiquement.

### Si le réseau ne remonte pas après redémarrage
Depuis la console physique :

```bash
sudo cp /root/interfaces.secours /etc/network/interfaces
sudo systemctl restart networking
ip -br addr show
```

Puis vérifier l'accès SSH depuis un poste client.

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
