# Déploiement — tryton

Fichiers de déploiement de l'environnement tryton.

## Contenu attendu

- `docker-compose.yml` : définition des services, images épinglées sur un tag explicite
- `.env.example` : variables attendues, sans valeur
- fichiers de configuration éventuels

## Mise en service

```bash
cp .env.example .env
# renseigner les valeurs
docker compose up -d
```

## Notes

Renseigner ici les particularités du déploiement : dépendances, ordre de démarrage,
étapes d'initialisation, points de blocage rencontrés.

La fiche de référence technique complète est dans `docs/fiches/`.
