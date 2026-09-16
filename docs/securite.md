# Sécurité du serveur POC

À compléter pendant l'étape 1 du plan projet.

## 1. Principe

Aucun port public exposé hors du point d'entrée du tunnel. SSH, interfaces ERP et
services techniques ne sont joignables que depuis le tunnel WireGuard.

## 2. Comptes système

| Élément | État |
|---|---|
| Utilisateur non privilégié créé | |
| Connexion root désactivée | |
| Authentification SSH par mot de passe désactivée | |
| SSH sur clé uniquement | |

## 3. WireGuard

**Port UDP retenu :**
**Plage d'adresses du tunnel :**

Procédure d'ajout d'un utilisateur au tunnel :

1.
2.
3.

Les clés privées ne figurent jamais dans ce dépôt.

## 4. Pare-feu OVH (Network Firewall)

| Règle | Protocole | Port | Source | Action |
|---|---|---|---|---|
| | | | | |

## 5. Pare-feu local (nftables)

Politique d'entrée : `DROP`.

Exceptions autorisées :

```
```

## 6. Durcissement

| Élément | État |
|---|---|
| Mises à jour de sécurité automatiques | |
| Services inutiles désactivés | |
| fail2ban | |
| Journalisation | |

## 7. Vérification

Scan externe réalisé le : 
Résultat attendu : aucun service détecté.

Résultat constaté :

## 8. Sauvegardes

| Élément | État |
|---|---|
| Backup Agent OVH activé | |
| Première sauvegarde vérifiée | |
| Fréquence et rétention | |
