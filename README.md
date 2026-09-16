# SI // ERP — Dépôt du POC (phase 1)

Dépôt de travail pour la phase 1 du projet ERP : déploiement et test comparatif de
quatre solutions (Tryton, Dokos, ERPNext, Axelor Community).

Documents de cadrage de référence, hors dépôt :
- `26-09-03_SI-ERP_Deploiement-environnements-test_V1.10.md`
- `26-09-03_SI-ERP_Premiers-tests-POC_V1.10.md`
- `26-09-15_SI-ERP_Plan-projet-POC_V1.01.md`

## Structure

```
infra/<erp>/     Fichiers Docker Compose, configuration de déploiement
docs/            Documentation d'installation, sécurité, décisions
docs/fiches/     Fiche de référence technique remplie, par ERP
docs/templates/  Modèles vierges à copier
tests/common/    Jeu de données commun aux quatre ERP
tests/<erp>/     Scripts de chargement et fiches de résultat
custom/<erp>/    Développements spécifiques, séparés du cœur applicatif
```

## Règles du dépôt

**Secrets.** Aucun mot de passe, token ou clé dans le dépôt. Les valeurs réelles vont
dans des fichiers `.env`, exclus par `.gitignore`. Chaque environnement dispose d'un
fichier `.env.example` documentant les variables attendues, sans valeur.

**Versions.** Aucune image Docker sur `latest`. Les tags et commits sont explicites et
relevés dans la fiche de référence de l'ERP concerné.

**Cœur applicatif.** Aucune modification directe du code cœur d'un ERP. Toute
personnalisation va dans `custom/<erp>/` ou dans un module séparé propre au framework.
Un blocage qui ne se résout que par une modification du cœur est un constat de test,
pas un problème à contourner.

**Documentation.** Toute manipulation non triviale est consignée dans `docs/` au fil de
l'eau. Le critère de réussite du déploiement impose qu'un environnement soit recréable
depuis ce dépôt : une action non documentée est considérée comme non faite.

**Données.** Aucune donnée réelle. Le jeu de données est fictif, décrit dans
`tests/common/jeu-de-donnees.md`.

## Commandes usuelles

```bash
git add -A
git commit -m "Description de ce qui a été fait et pourquoi"
git push
```

Les messages de commit servent à retrouver le contexte d'une modification plusieurs
semaines plus tard. Un message explicite vaut mieux qu'un message court.
