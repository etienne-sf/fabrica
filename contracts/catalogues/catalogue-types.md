# Catalogue — Types d'attributs, formats et valeurs par défaut dynamiques

> Registre des types, sous-types, formats de chaîne et valeurs par défaut dynamiques. Mécanisme et
> définitions : **ADR product-0037**. Catalogues **fermés**, extensibles par Fabrica. Tout scalaire
> personnalisé référence sa spécification par `@specifiedBy` ; aucune implémentation n'est développée
> dans Fabrica.

## Types et sous-types

| Type | Sous-type | JSON | Scalaire GraphQL | Spécification |
|---|---|---|---|---|
| `text` | `line` | chaîne | `String` | intégré |
| `text` | `multiline` | chaîne | `String` | intégré |
| `text` | `rich` | chaîne (balisage nettoyé) | `String` | intégré |
| `number` | `integer` | nombre (32 bits) | `Int` | intégré |
| `number` | `decimal` | chaîne | `Decimal` | https://scalars.graphql.org/chillicream/decimal — à vérifier |
| `boolean` | — | booléen | `Boolean` | intégré |
| `temporal` | `date` | chaîne `YYYY-MM-DD` | `LocalDate` | https://scalars.graphql.org/andimarek/local-date |
| `temporal` | `time` | chaîne ISO 8601 | `LocalTime` | https://scalars.graphql.org/chillicream/local-time |
| `temporal` | `datetime` | chaîne RFC 3339 (UTC) | `DateTime` | https://scalars.graphql.org/andimarek/date-time |
| `reference` | — | chaîne (`sys_id`) | `ID` + objet | intégré |
| `valuelist` | — | chaîne (code) | `String` | intégré |

## Formats de chaîne (`text/line`)

Vocabulaire aligné sur le mot-clé `format` de JSON Schema.

| Format | Scalaire GraphQL | Spécification |
|---|---|---|
| `email` | `EmailAddress` | RFC 5322 |
| `uri` | `URI` | https://scalars.graphql.org/chillicream/uri |
| `uuid` | `UUID` | https://scalars.graphql.org/chillicream/uuid |

## Valeurs par défaut dynamiques

| Valeur | Applicable à | Moment |
|---|---|---|
| `utilisateur_courant` | `reference` vers les utilisateurs | création |
| `date_de_creation` | `temporal/date` | création |
| `date_heure_de_creation` | `temporal/datetime` | création |

> **Réservés** : grands entiers (`long`), montants avec devise, durées, fichiers, expression régulière
> libre (`pattern`).
