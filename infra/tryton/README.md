# Déploiement — Tryton

**Boucle 4.1 du plan projet. Premier ERP testé.**

Tryton est un ERP modulaire écrit en Python, utilisant PostgreSQL. Son déploiement est
le plus simple des quatre : pas de workers, pas de Redis, pas de JVM. C'est pour cette
raison qu'il ouvre la série.

## Sources officielles

- Site : https://www.tryton.org/
- Documentation : https://docs.tryton.org/
- Téléchargements : https://downloads.tryton.org/
- Dépôt : https://github.com/tryton/tryton
- Image Docker : https://hub.docker.com/r/tryton/tryton

## Version retenue

Série **8.0**, version de long support sortie le 20 avril 2026.

L'image officielle publie chaque série sous son numéro de version. La variante suffixée
`-office` ajoute la conversion de documents OpenDocument vers d'autres formats, dont le
PDF : elle est nécessaire au test 13.

**À faire au moment de l'installation** : relever la dernière version corrective de la
série 8.0 et l'épingler précisément dans `.env`. Une vulnérabilité concernant l'accès
aux fichiers locaux lors du rendu d'un rapport HTML vers PDF a été corrigée en 2026.

**Note de calendrier** : la série 8.2 sort le 5 octobre 2026. Si elle est disponible au
moment du test 14 (montée de version), elle permettra de tester une montée de série
majeure, plus représentative qu'une simple version corrective. À arbitrer le moment venu.

## Mise en service

```bash
cp .env.example .env
# Renseigner DB_PASSWORD et vérifier HTTP_PORT et DOCKER_SUBNET

# Vérifier l'absence de conflit de sous-réseau avant de démarrer
docker network ls
docker network inspect <reseau> | grep Subnet

docker compose up -d postgres
docker compose logs -f postgres    # attendre que la base soit prête
```

### Initialisation de la base

L'initialisation active les modules et crée le compte administrateur. Elle se fait en
une seule commande, avant le premier démarrage du serveur.

```bash
docker compose run --rm trytond trytond-admin -d tryton --all -v
```

La commande demande le mot de passe du compte `admin`. Le choisir robuste et le
consigner dans le gestionnaire de secrets, jamais dans le dépôt.

Puis :

```bash
docker compose up -d
docker compose ps
curl -I http://127.0.0.1:8210
```

## Modules à activer pour les tests

Tryton n'installe que ce qu'on lui demande. Les modules ci-dessous couvrent les 14 tests.
Les noms sont donnés sous leur forme usuelle ; **vérifier leur disponibilité exacte en
série 8.0 dans la documentation avant activation**, la liste évolue d'une série à l'autre.

| Besoin | Module | Test concerné |
|---|---|---|
| Comptabilité générale | `account` | 10.1, 10.2 |
| Plan comptable français | localisation FR (à identifier) | 10.1, 10.2 |
| Achats fournisseurs | `purchase` | 4, 5, 6 |
| Factures | `account_invoice` | 6, 7 |
| Rapprochement achat / facture | `account_invoice_line_standalone`, `purchase_invoice_line_standalone` (à vérifier) | 5, 6 |
| Analytique | `analytic_account`, `analytic_purchase`, `analytic_invoice` | 3, 4 |
| Immobilisations | `account_asset` | 10.6 |
| Paiements et SEPA | `account_payment`, `account_payment_sepa` | 10.4 |
| Réception / BL | `stock`, `purchase` (réception) | 5 |
| Pièces jointes | `attachment` (cœur) | 4, 9 |

**Point d'attention — localisation française.** La complétude de la localisation FR
(plan comptable, TVA, FEC) est précisément ce que mesure le test 10. Ne pas présumer du
résultat : consigner ce qui vient du cœur, ce qui vient d'un module de localisation, et
ce qui manque.

**Point d'attention — poste de dépense.** Tryton dispose d'un module analytique
générique permettant plusieurs axes. Vérifier s'il permet de créer un axe « poste de
dépense » distinct du projet, saisi en en-tête et propagé aux lignes (test 4).

## Sauvegarde et restauration

À tester avant tout test métier, conformément au plan.

```bash
# Sauvegarde de la base
docker compose exec -T postgres pg_dump -U tryton -Fc tryton > sauvegardes/tryton_$(date +%F).dump

# Sauvegarde des fichiers applicatifs (pièces jointes)
docker run --rm -v erppoc-tryton_app-data:/data -v "$PWD/sauvegardes":/sauvegardes \
  alpine tar czf /sauvegardes/tryton_app_$(date +%F).tar.gz -C /data .
```

Restauration : recréer une base vide, puis `pg_restore`. Noter le temps constaté et les
commandes exactes dans la fiche de référence.

Le répertoire `sauvegardes/` est exclu par le `.gitignore`.

## Points à consigner dans la fiche de référence

- version exacte de l'ERP et du framework, tag de l'image, empreinte de l'image ;
- modules activés et leur provenance (cœur ou tiers) ;
- temps d'installation constaté ;
- particularités du modèle de données utiles aux tests suivants.
