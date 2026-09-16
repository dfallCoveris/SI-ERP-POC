# Préparation — Axelor Community

**Boucle 4.4 du plan projet. Quatrième ERP testé.**

**Statut : préparation. Les fichiers de déploiement seront écrits au moment de
l'installation, à partir des dépôts officiels.**

## Ce qu'est Axelor

Axelor Open Suite est une suite de gestion écrite en Java, reposant sur la plateforme
Axelor Open Platform, avec PostgreSQL comme base. Le déploiement met en jeu une
application Java packagée (WAR) servie par un conteneur de servlets.

C'est le plus lourd des quatre à déployer, d'où sa place en fin de série.

## Sources officielles

- Site : https://axelor.com/
- Suite applicative : https://github.com/axelor/axelor-open-suite
- Application web : https://github.com/axelor/open-suite-webapp
- Plateforme : https://github.com/axelor/axelor-open-platform

## Règle impérative — périmètre Community

**La phase 1 doit être réalisée uniquement avec le code public et Community réellement
utilisable sans licence commerciale.**

Ne jamais activer, même temporairement, une licence d'essai Pro ou Enterprise. Une
fonction validée sous licence d'essai fausserait entièrement la comparaison, puisque le
cadrage exige un ERP sans abonnement obligatoire.

Consigner explicitement, pour chaque composant installé :
- ce qui est public et sous licence AGPL ;
- ce qui relève d'une offre commerciale ;
- ce qui est simplement indisponible dans l'environnement testé.

C'est l'objet principal du test 1, et un critère potentiellement éliminatoire.

## Méthode de déploiement prévue

1. Identifier la version stable courante de l'Open Suite et la version de plateforme
   correspondante. Relever les tags précis.
2. Construire ou récupérer l'application selon la méthode documentée officiellement.
3. Déployer avec PostgreSQL, nom de projet `erppoc-axelor`, port sur `127.0.0.1`,
   sous-réseau dédié, base non publiée sur l'hôte.
4. Prévoir un dimensionnement mémoire JVM raisonnable : c'est l'environnement le plus
   consommateur des quatre.
5. Renseigner la fiche de référence avant tout test métier.

## Points de vigilance

**Temps de démarrage.** Une application Java de cette taille met plusieurs minutes à
démarrer, et davantage à la première initialisation de la base. Ne pas conclure trop
vite à un échec.

**Modules.** Axelor Open Suite est très large. N'activer que ce qui sert aux tests, et
noter ce qui est activé par défaut sans avoir été demandé.

**Extension sans modification du cœur (test 12).** Axelor propose son propre mécanisme
de modules et de studio. Vérifier ce qui relève du code public et ce qui suppose une
édition commerciale : la frontière est précisément le risque identifié sur cette
solution.

## Rappel

Aucune modification du cœur. Toute personnalisation va dans un module séparé, versionné
dans `custom/axelor/`.
