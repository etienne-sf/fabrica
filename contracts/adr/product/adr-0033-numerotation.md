# ADR-0033 — Numérotation : capacité `numérotée` et identifiant fonctionnel `number`

**Statut :** Proposé — 2026-07-11
**Famille :** **Produit** (`product/`). Capacité d'entité optionnelle (mécanisme : ADR-0025).
Formalise l'acquis (colonnes système ADR-0020 : `number` n'est **pas** une colonne système).
Générique (Principe IV).

---

## Contexte

`number` est l'**identifiant fonctionnel** d'un objet (le « numéro de ticket » — ex. `INC000123`) :
lisible, cité par les humains, **distinct** du `sys_id` (UUID technique, ADR-0020). Il n'existe que
si l'entité active la capacité **`numérotée`** — ce n'est donc pas une colonne système (qui, elle,
serait sur toute entité).

## Décision — capacité `numérotée`

- **Capacité optionnelle** (ADR-0025) : quand activée, l'entité porte un attribut **`number`**,
  **obligatoire et unique** — l'identifiant fonctionnel de l'objet.
- **Format** = **préfixe** + **N chiffres**. Le préfixe (trigramme/quadrigramme) est **unique par
  plateforme** et **immuable au MVP** (le changer imposerait une migration — réservé).
- **Nombre de chiffres** : **imposé globalement par Fabrica au MVP** (**défaut : 6**). Le
  paramétrage **par entité** est **réservé** (le global servira alors de **défaut** des nouvelles
  entités).

## Décision — génération des numéros

- **Un compteur par entité numérotée.** Ce compteur (le « prochain numéro ») est une **donnée de
  production** (il avance à chaque création en prod) — **hors métamodèle** (cohérent avec la
  frontière métamodèle / seed / données de production, tool-0001).
- **Démarre automatiquement à 1** au MVP.
- **Modifiable par l'administrateur, uniquement à la hausse** — baisser le compteur régénérerait des
  numéros déjà attribués (doublons). Garde-fou anti-doublon.
- Numéros **croissants, uniques, sans continuité garantie** : un numéro **réservé** (ex. à l'ouverture
  d'un objet) puis non utilisé laisse un **trou** — assumé (le confort « avoir le numéro tôt » prime
  sur la continuité).

## Hors périmètre (renvois)

- **Préfixe modifiable** (avec migration) ; **nombre de chiffres par entité** : réservés.
- `number` comme **forme d'URL** : réservé (ADR-0034 : URL par `sys_id` au MVP).
- Capacité / colonnes système : ADR-0025 / ADR-0020 (`number` n'est pas une colonne système).

## Alternatives écartées

- **`number` comme colonne système** : n'existe pas sur toute entité, et il est visible/fonctionnel.
  C'est l'attribut d'une capacité (ADR-0020).
- **Compteur dans le métamodèle** : c'est une donnée de production (il avance en prod). Hors git.
- **Compteur baissable** : régénérerait des doublons. À la hausse seulement.
- **Nombre de chiffres par entité au MVP** : un paramètre de plus. Global au MVP, par-entité réservé.
