# Jeu de données commun

**Version : V1.00**
**Date : 17/09/2026**

Ce fichier est la référence unique du jeu de données appliqué aux quatre ERP.
Il est figé avant la première installation. Toute évolution ultérieure est reportée
sur les environnements déjà créés, ou consignée comme écart en section 12.

## Règle de confidentialité

**Aucune donnée réelle.** Les noms de sociétés, de fournisseurs, de salariés et les
montants sont fictifs. En revanche, la **structure** est celle du groupe :
nomenclature des postes de dépense, circuit de validation, logique des sociétés,
régimes de TVA.

Les échanges métier ayant servi à construire ce document contiennent des noms de
personnes et des informations d'organisation. Ils ne sont pas versés dans ce dépôt.

## Sources

- Extractions Kalitics du 16/09/2026 (commandes, factures, écritures, grand livre).
- Réponses métier du 17/09/2026.
- `26-09-03_SI-ERP_Deploiement-environnements-test_V1.10.md`, section 6.

---

## 1. Postes de dépense

**Référentiel complet, tel qu'utilisé par le groupe.** C'est un axe analytique pur :
il sert au suivi de budget et de marge par affaire, pas à la comptabilité générale.
Les directeurs et chefs de projet raisonnent en postes lors de l'établissement d'un
budget de marché.

Correspondance stricte : un poste égale un compte comptable.

| Code | Libellé | Compte comptable |
|---|---|---|
| 01 | Vitrage | 60114000 |
| 02 | Remplissages divers, occultations | 60116000 |
| 03 | Alu. barre | 60110000 |
| 04 | Alu. assemblé | 60112000 |
| 05 | Acier barre | 60111000 |
| 06 | Acier assemblé | 60113000 |
| 07 | Ossature bois | 60119000 |
| 08 | Fabrication | 60430000 |
| 09 | Patte, pliage, précadre | 60115000 |
| 10 | Quincaillerie, visserie, étanchéité | 60223000 |
| 11 | Traitement de surface | 60118000 |
| 12 | Transport | 62410000 |
| 13 | Nacelle, manuscopique | 60117200 |
| 14 | Logistique chantier (grue, bung, échafaudage) | 60117000 |
| 15 | Pose, main d'œuvre | 60411000 |
| 16 | Sous-traitance | 60413000 |
| 17 | Étude | 60421000 |
| 18 | Prorata | 60480000 |
| 19 | Encadrement, déplacements | 62510119 |
| 20 | Aléas | 60490000 |

### Frais généraux

Les dépenses hors affaire n'utilisent pas cette nomenclature : le poste **est**
directement le compte comptable (61520000 Entretien sur biens immobiliers,
62263000 Honoraires, etc.).

C'est une convention voulue, pas un héritage. Le référentiel de l'ERP devra donc
supporter les deux natures, ou les distinguer explicitement.

### Nomenclature retenue pour les tests

Les 20 postes sont repris à l'identique, plus deux postes de frais généraux
(61520000 et 62263000) pour couvrir le second cas.

---

## 2. Sociétés

### Structure réelle (principe)

- Une holding détient cinq sociétés opérationnelles.
- Deux sociétés de type « MANCO » détiennent des parts, dont l'une réalise du chiffre
  d'affaires via un contrat de prestation avec une société opérationnelle.
- Une société immobilière détient les bâtiments ; son chiffre d'affaires n'est
  constitué que de loyers refacturés.
- Les charges de groupe sont centralisées chez la holding, qui refacture à chaque
  filiale (sauf la société immobilière et l'une des MANCO) une prestation de
  management fees, calculée selon le poids de chaque société. Plusieurs factures par
  an.
- Tous les exercices comptables sont alignés sur l'année civile.

### Sociétés de test

| Code | Nom (fictif) | Type | Rôle dans les tests |
|---|---|---|---|
| HOLD | Holding Test | Holding | Émet les refacturations de management fees |
| STA | Société Travaux A | Opérationnelle | Société principale des tests |
| STB | Société Travaux B | Opérationnelle | Cloisonnement des droits, sous-traitance interne |
| IMMO | Immo Test | Immobilière | Refacture des loyers à STA |

Quatre sociétés suffisent : elles couvrent la holding, deux opérationnelles et le cas
de la refacturation interne.

### Règles à respecter

- Une commande concerne **une seule société**.
- Un chantier peut concerner plusieurs sociétés, mais de façon distincte : chaque
  société reste indépendante, l'une pouvant être sous-traitante de l'autre.
- Toutes en euros.

---

## 3. Utilisateurs et droits

Le groupe compte une trentaine de groupes d'accès, chacun offrant une vue adaptée au
poste. Deux principes structurants :

- un utilisateur est rattaché à **une seule société**, celle pour laquelle il est
  salarié, et ne voit pas les données des autres ;
- les accès multi-sociétés existent mais restent exceptionnels (mise à disposition de
  salarié entre entités).

### Utilisateurs de test

| Identifiant | Rôle | Société | Rôle dans le circuit facture |
|---|---|---|---|
| admin | Administrateur | Toutes | Aucun |
| compta | Comptabilité | Toutes | Intègre la facture, affecte le valideur |
| acheteur-a | Acheteur | STA | Crée des commandes, premier valideur |
| chefprojet-a | Chef de projet | STA | Crée des commandes, premier valideur |
| conducteur-a | Conducteur de travaux | STA | Crée des commandes, premier valideur |
| dirachats-a | Directeur des achats | STA | N+1 de l'acheteur |
| dirtravaux-a | Directeur des travaux | STA | N+1 du conducteur |
| directeur-b | Directeur | STB | Cloisonnement : ne doit rien voir de STA |
| multi-ab | Chef de projet | STA + STB | Cas exceptionnel du salarié mis à disposition |

### Contrôles attendus

- Un acheteur ne peut pas saisir de facture de vente.
- Un chef de projet ne peut pas supprimer une facture fournisseur.
- `directeur-b` ne voit aucune donnée de STA.
- `multi-ab` voit les deux sociétés, et uniquement celles-là.

---

## 4. Circuit de validation des factures fournisseurs

### Circuit réel

1. La facture arrive, aujourd'hui par la plateforme de facturation électronique.
2. La comptabilité l'intègre et l'**affecte manuellement** à la personne qui a passé
   la commande (acheteur, chef de projet ou conducteur de travaux).
3. Cette personne valide : premier valideur.
4. La facture part ensuite chez le **supérieur hiérarchique du premier valideur**.
5. **Aucun seuil de montant** ne modifie le circuit.

### Refus et mise en attente

- **Refus** : le valideur doit demander un avoir au fournisseur. Depuis la
  facturation électronique, le refus est officiel et la facture est automatiquement
  renvoyée au fournisseur via les plateformes agréées respectives.
- **Mise en attente** : la facture est placée dans un menu dédié, ou simplement non
  validée en attendant la réponse du fournisseur.

### Validation par lots

**Non souhaitée par le métier.** Position explicite : valider plusieurs factures
simultanément conduirait à des validations à la hâte, sans analyse réelle.

Voir section 11 : divergence avec le test 8 du document de cadrage.

### Facture sans commande

Cas résiduel, limité à certains frais généraux de direction. **À partir de janvier
2027, chaque facture de frais généraux aura son bon de commande.**

Le jeu de données conserve un cas de facture sans commande pour tester la mécanique,
mais il ne constitue pas une exigence cible.

### Besoin non couvert aujourd'hui

Le valideur de second niveau est déterminé par le rattachement hiérarchique de
l'utilisateur, et non par le contexte de la commande. Or certaines personnes ont deux
casquettes : rattachées à un service, elles interviennent aussi sur des affaires
relevant d'un autre responsable. Le bon valideur n'est alors pas celui que le système
désigne.

**À tester explicitement** (test 7) : l'ERP permet-il de déterminer le second
valideur selon le contexte (société, chantier, type de dépense) plutôt que selon le
seul rattachement de l'utilisateur ?

`multi-ab` sert à ce cas.

---

## 5. Chantiers

### Format
`AA-NNN` : année sur deux chiffres, numéro séquentiel. Exemples réels : 22-261,
26-610, 26-294.

Dans les écritures comptables, le code apparaît concaténé : `220261`, `260610`.

### Informations d'en-tête attendues

Toutes jugées indispensables par le métier :

| Information | Remarque |
|---|---|
| Client | Distinct du nom de l'affaire |
| Nom de l'affaire | |
| Type de marché | Public ou privé |
| Dates | Début, fin prévisionnelle |
| Montant de marché | |
| Chef de projet | |
| Responsable d'études | |
| Acheteur | |
| Conducteur de travaux | |
| Marge en temps réel | Restitution, pas saisie |

Quatre rôles distincts sont donc rattachés à un chantier, pas un responsable unique.
Point à affiner côté métier.

### Chantiers de test

| Code | Nom (fictif) | Société | Type | Particularité |
|---|---|---|---|---|
| 26-001 | Résidence Test A | STA | Privé | Chantier principal |
| 26-002 | Groupe scolaire Test | STB | Public | Cloisonnement |
| 26-003 | Façade Test C | STA | Privé | STB intervient en sous-traitance |

---

## 6. Fournisseurs

| Code | Nom (fictif) | Type | Particularité |
|---|---|---|---|
| FOURN-001 | Alu Test | Matériaux | Commandes multi-lignes, poste 03 |
| FOURN-002 | Pose Test | Sous-traitant | Autoliquidation BTP, poste 16 |
| FOURN-003 | Euro Test | Européen | TVA intracommunautaire, sans TVA sur facture |
| FOURN-004 | Transport Test | Transporteur | Poste 12 |
| FOURN-IT | Info Test | Frais généraux | Poste = compte 62263000 |
| FOURN-SERV | Service Test | Prestataire de services | TVA sur encaissements à l'achat |

---

## 7. Comptes comptables

### Fournisseurs et clients
- 401 avec comptes auxiliaires par fournisseur (observé dans les extractions).
- 411 pour les clients.

### Charges
Les 20 comptes de la section 1, plus 61520000 et 62263000 pour les frais généraux.

### TVA
- 44566200 : TVA déductible sur biens.
- 44526 et 44527 : autoliquidation intracommunautaire (débit et crédit).
- Comptes d'autoliquidation sous-traitance BTP à créer.
- TVA collectée sur encaissements.

### Immobilisations
Un compte d'immobilisation et son compte d'amortissement.

### Lettrage
**Toute la classe 4 est lettrée**, pas seulement 401 et 411 :
- 42 : salaires, lettrés au paiement ;
- 43 : cotisations sociales, lettrées au prélèvement ;
- jusqu'aux comptes 49.

Le jeu de données prévoit au minimum un cas 401, un cas 411 et un cas 42.

---

## 8. TVA

| Régime | Application |
|---|---|
| Taux 5,5 %, 10 %, 20 % | Tous utilisés |
| Autoliquidation sous-traitance BTP | Sous-traitants |
| TVA intracommunautaire | Fournisseurs européens, même schéma que l'autoliquidation |
| TVA sur les encaissements | **Ventes** : régime du groupe |
| TVA sur les débits | **Achats** : fournisseurs de biens |
| TVA sur les encaissements | **Achats** : fournisseurs de services |

Le régime mixte à l'achat (débits pour les biens, encaissements pour les services)
est le point le plus exigeant. À vérifier explicitement dans le test 10.7.

---

## 9. Documents à créer

Montants volontairement ronds, sauf là où le cas de test impose un écart précis.

### 9.1 Commandes fournisseurs

| Réf. | Société | Chantier | Fournisseur | Contenu | Montant HT |
|---|---|---|---|---|---|
| CF-001 | STA | 26-001 | FOURN-001 | 4 lignes, poste 03 | 10 000 € |
| CF-002 | STA | 26-001 | FOURN-002 | 2 lignes, poste 16, autoliquidation BTP | 8 000 € |
| CF-003 | STB | 26-002 | FOURN-001 | 2 lignes dont une avec pièce jointe | 5 000 € |
| CF-004 | STA | 26-001 | FOURN-003 | 1 ligne, fournisseur européen | 3 000 € |
| CF-005 | STA | 26-003 | FOURN-004 | Transport, poste 12 | 1 000 € |
| CF-006 | STA | 26-001 | FOURN-001 | Commande à corriger après dépassement | 2 000 € |

**Point de test sur CF-001 et CF-005.** La politique actuelle impose un poste de
dépense unique par commande. Le référentiel comporte pourtant un poste 12 Transport
et un poste 20 Aléas, qui correspondent à des natures pouvant apparaître dans une
commande dont l'objet principal relève d'un autre poste. Vérifier ce que permet
l'ERP : poste en en-tête propagé aux lignes, mais modifiable ligne à ligne.

### 9.2 Factures fournisseurs

| Réf. | Commande | Montant HT | Objectif du test |
|---|---|---|---|
| FF-001 | CF-001 | 6 000 € | Facture partielle, commande reste ouverte |
| FF-002 | CF-001 | 4 000 € | Seconde facture, commande soldée |
| FF-003 | CF-006 | 2 500 € | Dépassement : impose la correction de la commande |
| FF-004 | CF-002 | 8 000 € | Autoliquidation BTP |
| FF-005 | CF-004 | 3 000 € | Fournisseur européen, sans TVA, deux écritures de TVA |
| FF-006 | aucune | 800 € | Facture sans commande, frais généraux |
| FF-007 | CF-003 | 5 000 € | Société STB, cloisonnement |

### 9.3 Autres documents

- **Avoir fournisseur** rattaché à FF-001, montant 500 € HT.
- **Immobilisation** : véhicule de fonction, immobilisé pour son montant TTC (la TVA
  n'est pas récupérable sur les véhicules de fonction), durée 5 ans, amortissement
  linéaire.
- **Refacturation interne** : une facture de management fees émise par HOLD vers STA,
  et une facture de loyer émise par IMMO vers STA.
- **Jeu d'écritures** permettant de produire bilan, compte de résultat et FEC,
  réparties sur deux exercices pour la comparaison N / N-1.
- **Cas de lettrage** : un 401, un 411, un compte 42 (salaire lettré au paiement).

---

## 10. Ordre de création

1. Sociétés
2. Plan comptable et journaux
3. Postes de dépense (axe analytique)
4. Utilisateurs, rôles et droits
5. Fournisseurs
6. Chantiers
7. Commandes
8. Factures et avoir
9. Immobilisation
10. Écritures complémentaires et lettrage

---

## 11. Impacts sur les tests du POC

Trois écarts entre les documents de cadrage et le fonctionnement réel, à arbitrer.

### 11.1 Test 8 — Validation par lots

Le cadrage demande de tester la validation par lots pour les N+1. Le métier ne la
souhaite pas, et argumente : risque de validation sans analyse.

**Le test reste à mener** (il mesure une capacité de l'ERP), mais son résultat ne
devrait pas être éliminatoire. À confirmer avec le tuteur.

### 11.2 Facturation électronique — déjà en production

Le cadrage la présente comme une évolution future via une Plateforme Agréée externe.
Elle est déjà en service : les factures arrivent par ce canal et les refus repartent
par ce canal.

Ce n'est donc pas une fonction à prévoir, mais un **prérequis** de la solution cible.
À intégrer au périmètre de la phase 2.

### 11.3 Test 7 — Valideur selon le contexte

Besoin non couvert aujourd'hui, identifié en section 4. Un ERP capable de déterminer
le second valideur selon le contexte plutôt que selon le seul rattachement
hiérarchique constituerait un gain net.

À ajouter explicitement aux points à vérifier du test 7.

### 11.4 Tests 10.3 et 10.7 — périmètre élargi

- **Lettrage** : toute la classe 4, pas seulement 401, 411 et 467.
- **TVA** : trois taux, autoliquidation BTP, intracommunautaire, sur encaissements
  aux ventes, et régime mixte aux achats selon biens ou services.

---

## 12. Écarts constatés par ERP

Quand un ERP ne permet pas de reproduire exactement ce jeu de données, consigner ici
l'écart et sa raison, plutôt que d'adapter silencieusement les données.

| ERP | Élément concerné | Écart | Raison |
|---|---|---|---|
| | | | |

---

## 13. Points restant à préciser

1. Informations d'en-tête d'un chantier : à affiner côté métier.
2. Comptes exacts d'autoliquidation sous-traitance BTP.
3. Fichier de calcul des management fees : structure, si elle doit être reproduite.
4. Colonnes manquantes dans les extractions actuelles (poste de dépense sur facture,
   équivalent compte comptable, montant payé HT, date d'échéance) : besoins de
   restitution à couvrir par l'ERP.

---

## 14. Historique des versions

| Version | Date | Évolution |
|---|---|---|
| V1.00 | 17/09/2026 | Création, à partir des extractions du 16/09 et des réponses métier du 17/09. |
