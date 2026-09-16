# Préparation — Dokos

**Boucle 4.2 du plan projet. Deuxième ERP testé.**

**Statut : préparation. Les fichiers de déploiement seront écrits au moment de
l'installation, à partir du dépôt officiel.**

## Ce qu'est Dokos

Dokos est un fork français d'ERPNext, reposant sur le framework Frappe (renommé Dodock
dans le fork). L'écosystème est le même : base MariaDB, Redis, workers de tâches de
fond, ordonnanceur, et une notion de « site » propre à Frappe.

Conséquence : **le déploiement ne s'écrit pas à la main**. On part du dépôt de
déploiement officiel et on l'adapte. Un Compose improvisé produirait une installation
non conforme et non reproductible.

## Sources officielles

- Documentation : https://doc.dokos.io/
- Organisation : https://gitlab.com/dokos
- Application : https://gitlab.com/dokos/dokos
- Framework : https://gitlab.com/dokos/dodock

## Méthode de déploiement prévue

1. Identifier le dépôt de déploiement conteneurisé recommandé par la documentation
   officielle, et la version de série correspondante.
2. Relever les versions exactes de Dokos et de Dodock, ainsi que le tag ou le commit.
3. Adapter la configuration : nom de projet `erppoc-dokos`, port publié sur
   `127.0.0.1` uniquement, sous-réseau dédié, aucune base publiée sur l'hôte.
4. Créer le site Frappe et installer l'application.
5. Renseigner la fiche de référence avant tout test métier.

Documenter séparément toute application additionnelle utilisée pour les workflows, les
factures entrantes ou la localisation.

## Points de vigilance propres à Dokos

**Poste de dépense (test 4).** Il doit être créé comme dimension comptable ou
référentiel dédié. Le champ **Entrepôt ne doit pas être détourné** : sa sémantique et
ses dépendances appartiennent au stock. À vérifier : saisie en en-tête, propagation
automatique sur les lignes, conservation jusqu'à la facture puis jusqu'aux écritures,
filtrage, disponibilité par API, et comportement si une ligne tente un autre poste.

**Article imposé (test 4).** Si un Item est structurellement obligatoire sur une ligne
de commande, tester un article technique renseigné automatiquement et invisible pour
l'utilisateur. Mesurer les conséquences sur l'ergonomie, le reporting, la réception, la
facture et la maintenance.

**Masquage du domaine Stock (test 4).** Le stock n'est pas un besoin métier du groupe.
Vérifier qu'on peut masquer les menus et retirer les droits associés sans supprimer les
briques techniques nécessaires au cycle achats, et sans modifier le cœur. Le cycle
commande vers facture doit rester fonctionnel, et l'absence de bon de livraison possible.

**Écarts avec l'upstream (étape 5).** Consigner au fil de l'eau les DocTypes et champs
spécifiques à Dokos, absents d'ERPNext. Ce relevé alimente directement le test de
réversibilité.

## Rappel

Aucune modification du cœur. Toute personnalisation va dans une application Frappe
séparée, versionnée dans `custom/dokos/`.
