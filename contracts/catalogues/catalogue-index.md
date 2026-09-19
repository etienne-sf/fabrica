# Index des catalogues — Fabrica

> Les **catalogues** sont les registres des familles fermées de Fabrica (motif : constitution
> § Catalogues). Chaque catalogue **renvoie** aux ADR qui décident, ou **décrit** les entrées qui ne
> portent pas de décision — jamais de duplication. Ils sont destinés à devenir des **données système
> réflexives** (interrogeables en RUN). Source de vérité : ces fichiers (git). À vérifier contre le
> dépôt.

| Catalogue | Contenu | Mécanisme / ADR |
|---|---|---|
| `catalogue-capacites.md` | Capacités d'entité (obligatoires : colonnes-système, actif ; optionnelles : à-états, numérotée ; pressenties : auditée, journalisée, affectable, commentable, hiérarchique) | **ADR-0025** (mécanisme) |
| `catalogue-effets.md` | Effets (interdire_modification, rendre_obligatoire, vider, masquer) | **ADR-0006** |
| `catalogue-natures-acl.md` | Natures d'ACL (`donnée:lecture/…/suppression`, `fonctionnalité:*`) | **ADR-0012** |
| `catalogue-canaux.md` | Régimes d'application des effets (donnée / formulaire) | **ADR-0006** |

## Règle commune (constitution § Catalogues)
- **Fermé** : le projet **active/référence**, il n'**invente** pas. Extensible **par Fabrica**.
- Une entrée qui **porte une décision** → **ADR** (le catalogue renvoie). Sinon → **décrite** ici.
- Un catalogue ne contient **jamais** de décision : registre navigable, pas source concurrente des ADR.

## Catalogues pressentis (non créés)
- Conditions de l'axe ligne (quand l'ADR axe ligne / RLS sera écrit).
- Types de points d'entrée (quand l'ADR scripts sera écrit).
- Familles de natures d'ACL futures (`api:*`, `rapport:*`).
