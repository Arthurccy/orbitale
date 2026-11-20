# Orbitale — API RESTful de billetterie & contrôle d'accès

Ce dépôt contient les spécifications d'architecture et les livrables de conception de l'API Orbitale (M2 Web Services).

## Livrables de la Séance 1
- [`docs/notes_s1.md`](docs/notes_s1.md) : Observations du TP guidé (anatomie HTTP, cache ETag, erreurs RFC 9457, pagination, négociation de contenu et échelle de Richardson).
- [`docs/ressources.md`](docs/ressources.md) : Modèle de ressources RESTful d'Orbitale (ressources métier, cardinalités Mermaid, table des endpoints `/v1/...`, justifications d'architecture).

## Conventions architecturales
- **Niveau 2 de Richardson** avec verbes HTTP stricts et codes de statut précis.
- URI orientées ressources (noms au pluriel, aucun verbe dans l'URI, préfixées par la version `/v1/`).
- Gestion standardisée des erreurs via la **RFC 9457** (`application/problem+json`).
