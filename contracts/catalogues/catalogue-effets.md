# Catalogue — Effets

> Registre des effets. Mécanisme et définitions : **ADR-0006** (effets multi-canaux). Application
> par le **moteur des effets** (ADR-0016). Catalogue fermé, extensible par Fabrica. Renvoi, pas
> duplication.

| Effet | Sens | Décision |
|---|---|---|
| `interdire_modification` | modification refusée (grisé en formulaire) | **ADR-0006** |
| `rendre_obligatoire` | valeur requise | **ADR-0006** |
| `vider` | efface la donnée (déclenché aussi par conditionnement de liste) | **ADR-0006** |
| `masquer` | caché (présentation ; donnée transmise) | **ADR-0006** |

> Chaque effet peut porter des **prérequis de capacités** (ADR-0025) : il n'est proposé au
> paramétrage d'une entité que si elle a les capacités requises.
