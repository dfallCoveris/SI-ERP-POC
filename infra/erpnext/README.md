# Préparation — ERPNext

**Boucle 4.3 du plan projet. Troisième ERP testé.**

**Statut : préparation. Les fichiers de déploiement seront écrits au moment de
l'installation, à partir du dépôt officiel.**

## Ce qu'est ERPNext

ERPNext est l'ERP open source construit sur le framework Frappe, dont Dokos est un fork.
Même architecture : MariaDB, Redis, workers, ordonnanceur, notion de site.

La boucle ERPNext bénéficie directement de l'apprentissage fait sur Dokos : le socle est
commun, les écarts sont ciblés. C'est la raison de cet ordre dans le plan.

## Sources officielles

- Produit : https://frappe.io/erpnext
- Application : https://github.com/frappe/erpnext
- Framework : https://github.com/frappe/frappe
- Déploiement conteneurisé : https://github.com/frappe/frappe_docker

## Méthode de déploiement prévue

Partir de `frappe_docker`, qui est le déploiement de référence, et l'adapter :

1. Choisir la variante adaptée à un environnement de test mono-machine.
2. Épingler les versions d'ERPNext et de Frappe (tag ou commit), jamais latest.
3. Nom de projet `erppoc-erpnext`, port sur `127.0.0.1`, sous-réseau dédié, aucune base
   publiée sur l'hôte.
4. Créer le site et installer l'application.
5. Renseigner la fiche de référence avant tout test métier.

## Points de vigilance propres à ERPNext

**Localisation française (test 10).** C'est le point le plus important à documenter.
Distinguer précisément :
- ce qui vient du cœur d'ERPNext ;
- ce qui vient d'une application de localisation France ;
- la licence et le mainteneur de chaque dépendance ;
- ce qui manque et devrait être développé.

Le cadrage interdit de redévelopper une fonction comptable réglementaire structurante :
c'est un critère potentiellement éliminatoire.

**Poste de dépense et article imposé (test 4).** Mêmes questions que sur Dokos. Les
constats faits sur Dokos servent d'hypothèses de départ, mais doivent être revérifiés :
le fork a pu diverger.

**Seconde instance pour la réversibilité (étape 5).** Si Dokos reste candidat, une
instance ERPNext de même génération doit être disponible en parallèle pour le test
d'import. Les 32 Go de la machine le permettent sans arbitrage. Prévoir un port et un
sous-réseau distincts de l'instance de test principale.

## Rappel

Aucune modification du cœur. Toute personnalisation va dans une application Frappe
séparée, versionnée dans `custom/erpnext/`.
