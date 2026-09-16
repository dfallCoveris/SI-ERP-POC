# Installation — socle technique

À compléter au fil de l'eau pendant l'étape 2 du plan projet.

## 1. Serveur

| Élément | Valeur |
|---|---|
| Offre | Kimsufi KS-5 |
| Processeur | Intel Xeon E3-1270 v6, 4c/8t |
| Mémoire | 32 Go DDR4 ECC |
| Stockage | 2 × 450 Go SSD NVMe, RAID logiciel |
| Système | Debian (version à renseigner) |
| Datacenter | |
| Date de mise en service | |

## 2. Docker

Versions installées :

```
docker --version
docker compose version
```

## 3. Arborescence

```
/erp-poc/
  tryton/
  dokos/
  erpnext/
  axelor/
```

## 4. Attribution des ports et sous-réseaux

| ERP | Port hôte | Sous-réseau Docker | Commentaire |
|---|---|---|---|
| Tryton | | | |
| Dokos | | | |
| ERPNext | | | |
| Axelor | | | |
| ERPNext (réversibilité) | | | |
| Reverse proxy | | | |

Règle : les interfaces ERP ne sont pas publiées sur l'interface publique. Elles écoutent
sur l'adresse du tunnel ou sur `127.0.0.1`, et sont atteintes via le reverse proxy.
Aucune base de données n'est publiée sur l'hôte.

## 5. Reverse proxy

Configuration par fichier, versionnée dans `infra/`.

## 6. Gestion des secrets

Un fichier `.env` par environnement, jamais committé. Un `.env.example` documente les
variables attendues.

## 7. Sauvegarde et restauration

Procédure de sauvegarde des volumes :
```bash
```

Procédure de restauration :
```bash
```

Backup Agent OVH : état de configuration, fréquence, rétention.
