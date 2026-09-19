# Catalogue — Natures d'ACL

> Registre des natures d'ACL. Mécanisme et définitions : **ADR-0012** (modèle d'autorisation).
> Convention de nommage : `famille:déclinaison`. Catalogue fermé, extensible par Fabrica.

| Nature | Sens | Décision |
|---|---|---|
| `donnée:lecture` | lire l'attribut/l'entité cible | **ADR-0012** |
| `donnée:lecture_écriture` | lire + modifier | **ADR-0012** |
| `donnée:lecture_écriture_suppression` | lire + modifier + supprimer | **ADR-0012** |
| `fonctionnalité:exécuter` | exécuter une fonctionnalité livrée (se combine aux droits donnée) | **ADR-0012** |

> Familles futures (`api:*`, `rapport:*`…) : même convention, à ajouter par Fabrica.
