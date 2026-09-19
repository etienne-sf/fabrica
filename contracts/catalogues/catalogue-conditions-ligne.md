# Catalogue — Conditions de l'axe ligne

> Registre des conditions utilisables par les ACL de ligne. Mécanisme et définitions : **ADR-0026**
> (autorisation par ligne). Catalogue **fermé**, extensible par Fabrica. Renvoi, pas duplication.
> Limite dure : toute condition doit être **exprimable en prédicat SQL/RLS**.

## Familles

| Famille | Sens | Décision |
|---|---|---|
| `valeur` | un champ comparé à une constante (ex. `statut = brouillon`) | **ADR-0026** |
| `relation-utilisateur` | un champ comparé à une valeur d'axe de gouvernance de l'utilisateur courant (ex. le groupe de la ligne ∈ mes groupes) — cas dominant | **ADR-0026** |

## Axes de gouvernance (catalogue fermé)

| Axe | Résout, pour l'utilisateur courant | Décision |
|---|---|---|
| `utilisateur` | l'identité courante (propriétaire/créateur) | **ADR-0026** |
| `groupe` | les groupes de l'utilisateur | **ADR-0026** |

> `rôle` écarté comme axe au démarrage (sans valeur distincte du groupe).
