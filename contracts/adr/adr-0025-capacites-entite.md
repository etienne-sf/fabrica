# ADR-0025 — Capacités d'entité (mécanisme)

**Statut :** Proposé — 2026-07-11
**Portée :** décision du **cœur** (Fabrica), générique (Principe IV). **Méta-mécanisme** : définit ce
qu'est une capacité et comment elle s'active. La **liste** des capacités vit dans
`contracts/catalogues/catalogue-capacites.md` (motif « Catalogues », constitution). À éclater en
`contracts/adr/0025-capacites-entite.md`.

---

## Contexte

`à-états` (ADR-0019), `numérotée`, `actif`, les colonnes système… suivent un même schéma : une
**propriété déclarative d'une entité active un paquet cohérent** (colonnes, contraintes,
comportements). Plutôt qu'un catalogue de cas particuliers, on pose le **mécanisme générique** :
la **capacité d'entité**. Cet ADR définit le mécanisme ; il **n'énumère pas** les capacités (elles
vivent dans le catalogue, chacune renvoyant à son ADR si elle porte une décision).

## Décision — ce qu'est une capacité

Une **capacité** est un paquet activable par entité, qui peut apporter :
- des **colonnes / attributs** (ex. `state`, `number`, `actif`) — dont les **noms sont réservés** ;
- des **contraintes** (unicité, obligation…) ;
- des **comportements** (crochets before/after, effets débloqués — ci-dessous) ;
- des **structures liées** (ex. table de transitions, table d'historique) ;
- des **points d'extension** (au sens ADR-0013).

## Décision — forme d'activation uniforme

Une entité déclare une **liste de capacités activées**, chacune avec un **bloc de paramétrage** dont
le **schéma est propre à la capacité** (le graphe pour `à-états`, préfixe+chiffres pour `numérotée`…).
Fabrica impose la **coquille commune** (entité → liste de (capacité, paramétrage)) ; chaque capacité
définit le **schéma de son bloc**. Même motif que la charge utile structurée par nature des ACL
(ADR-0012) : on uniformise la **structure d'accueil**, pas le contenu.

## Décision — capacités système vs optionnelles

- **Capacités système** (présentes sur **toute** entité, non supprimables) : au démarrage,
  **colonnes système** (ADR-0020), **`actif`** (ADR-0023), **`valeur-affichage`** (ADR-0027) et
  **`audit`** (ADR-0028). Elles **réservent leurs noms d'attributs** (`sys_*`, `actif`) — un projet
  ne peut pas les redéfinir (préfixes réservés, ADR-0017). Une capacité système est un **organe
  toujours présent** dont le **comportement à l'exécution est déterminé par sa configuration**
  (souvent inerte par défaut ; ex. `audit` ne fait rien tant qu'aucun niveau de traçabilité n'est
  déclaré). La configuration ne crée/supprime pas la capacité — elle règle ce qu'elle **fait**.
- **Capacités optionnelles** : activées par le projet (`à-états`, `numérotée`, `historiser`…).

## Décision — indépendance des capacités, prérequis au niveau des effets

- Les **capacités sont indépendantes entre elles** : activer l'une n'exige jamais une autre, aucun
  ordre d'activation, aucune dépendance croisée.
- La contrainte de combinaison ne vit **pas** entre capacités mais dans les **effets** : un effet
  peut déclarer des **prérequis de capacités**, et n'est **proposé au paramétrage** que si l'entité
  les satisfait. Exemple : « à la transition *validation*, affecter à tel groupe » est un effet qui
  requiert **`à-états`** (pour la transition) **et** **`affectable`** (pour l'affectation). Le lien
  affectable↔état ne disparaît pas — il se **déplace** vers la disponibilité de l'effet.
- Conséquence : le **catalogue d'effets est filtré par les capacités de l'entité** (« quels effets
  puis-je paramétrer ici ? → ceux dont les prérequis de capacités sont satisfaits »). *Amende
  l'ADR-0006* (un effet porte des prérequis de capacités). Les capacités **débloquent** des effets ;
  elles ne se connaissent pas.

## Décision — catalogue fermé, connu de Fabrica

- Le catalogue des capacités est **fermé** : le projet **active**, il n'**invente** pas (Fabrica doit
  savoir câbler chaque capacité). Extensible **par Fabrica** (ses versions).
- La **liste** vit dans `contracts/catalogues/catalogue-capacites.md` (registre : renvoi vers l'ADR pour les
  capacités qui portent une décision, description sinon), destinée à devenir une **donnée système
  réflexive** (ADR-0019, métamodèle en table lecture seule).

## Lien avec DICT (constitution)

Certaines capacités réalisent des critères DICT (`auditée` → Traçabilité, chiffrement →
Confidentialité, droits → Confidentialité). Comme les capacités actives d'une entité sont
déclarées et lisibles, elles **contribuent à la mesurabilité DICT** exigée par la constitution :
« cette entité est-elle traçable ? → active-t-elle `auditée` ? » se lit dans le catalogue/le
métamodèle réflexif.

## Alternatives écartées

- **Énumérer les capacités dans cet ADR** : forcerait à l'amender à chaque nouvelle capacité. La
  liste vit dans le catalogue, séparément.
- **Dépendances dures entre capacités** : créeraient ordre d'activation et fragilité. Remplacées
  par les prérequis de capacités au niveau des effets.
- **Paramétrage commun imposé** : écraserait les spécificités (le graphe d'états n'a rien à voir
  avec un préfixe de numéro). Remplacé par « coquille commune + schéma propre à chaque capacité ».
- **Capacités inventables par le projet** : Fabrica ne saurait pas les câbler. Catalogue fermé.

## Conséquences

- L'anatomie (ADR-0013) est complétée : une entité = ses attributs + ses **capacités** (obligatoires
  + optionnelles activées).
- L'ADR-0006 gagne la notion de **prérequis de capacités** sur les effets.
- Les capacités déjà identifiées (`à-états`, `numérotée`, `actif`, colonnes système) et pressenties
  (`auditée`, `journalisée`, `affectable`, `commentable`…) sont listées au catalogue, chacune renvoyant
  à son ADR ou décrite si trop légère pour un ADR.
