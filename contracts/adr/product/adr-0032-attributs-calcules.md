# ADR-0032 — Attributs calculés et valeur d'affichage (display value)

**Statut :** Proposé — 2026-07-11
**Famille :** **Produit** (`product/`). Premier client du **mécanisme de script** (ADR-0031).
Réalise la capacité obligatoire `valeur-affichage`. Générique (Principe IV).

---

## Contexte

Un **attribut calculé** dérive sa valeur d'autres attributs. Son premier usage est la **valeur
d'affichage** (*display value*) — représentation lisible d'un objet (« prénom nom »), consommée par
les champs Référence, listes, historisation (ADR-0030, 0027).

## Décision — attribut calculé = script pur sur la ligne courante

- Un attribut calculé est réalisé par un **script** (ADR-0031) : **pur**, sur les **attributs de la
  ligne courante uniquement** (aucune jointure), retour **texte** au MVP. Ex. `prénom + " " + nom`.
- **Pas de catalogue d'opérations maison** : les fonctions pures de TypeScript suffisent (ADR-0031).
- **Formules pures, sans dépendance au temps** : pas de « date du jour » dans un attribut matérialisé
  (sinon il périmerait silencieusement — un « âge » calculé se fausserait le lendemain sans recalcul).
  Les calculs temporels attendront le mode **virtuel** (réservé).

## Décision — matérialisé (le virtuel est réservé)

- Au MVP, un attribut calculé est **matérialisé** : il a une **colonne**, sa valeur est **stockée**
  et **recalculée à la modification** via le **moteur des effets** (ADR-0016 — un calculé est comme
  une règle « quand une source change, recalculer » ; les cascades calculé→calculé sont gérées par le
  **point fixe**).
- Le mode **virtuel** (calculé à la lecture, non stocké — pas de dénormalisation) est **réservé**
  (post-MVP). Choisi ainsi pour simplifier le MVP.

## Décision — display value : dénormalisation justifiée

- `valeur-affichage` est une **capacité obligatoire** (ADR-0025 / catalogue) de toute entité :
  représentation d'affichage d'un objet.
- Réalisée soit par un **attribut existant désigné** (le `name`), soit par un **attribut calculé**
  (« prénom nom »).
- **Matérialisée** — parce qu'elle est **cherchée massivement** : l'autocomplétion du champ Référence
  (ADR-0030) filtre *par display value* (« tape Dup → Dupont »), ce qui exige qu'elle soit
  requêtable/indexable en base. C'est une **dénormalisation justifiée par un besoin mesuré** (la
  recherche) — donc **conforme** à « pas de dénormalisation *par défaut* » (constitution) : ici elle
  est justifiée, pas par défaut.

## Hors périmètre (renvois)

- **Mode virtuel** (calcul à la lecture) ; **calculs temporels** ; **cross-relation** (calcul
  traversant une relation) ; **retour non-texte** : réservés.
- **Mécanisme de script** : ADR-0031.
- **Recalcul** : moteur des effets (ADR-0016).
- **Consommateurs** du display value : rendu IHM (ADR-0030), historisation (ADR-0027).

## Alternatives écartées

- **Catalogue d'opérations de calcul maison** : réimplémente TypeScript (ADR-0031). Script pur.
- **Virtuel par défaut** : ne servirait pas la recherche du display value (autocomplétion) et
  compliquerait le MVP. Matérialisé au MVP, virtuel réservé.
- **Calcul cross-relation / temporel au MVP** : jointures / impureté. Réservés.
