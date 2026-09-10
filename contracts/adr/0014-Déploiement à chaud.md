# ADR-0014 — Déploiement à chaud : instance, versions, rolling update

**Statut :** Proposé — 2026-07-11
**Portée :** décision du **cœur** (Fabrica), générique (Principe IV). Contrainte transverse qui
s'appuie sur l'additivité déjà décidée (expand/contract, ADR-0002/Principe VI) et la contraint
pour la suite (montées de version, personnalisation). À éclater en
`contracts/adr/0014-deploiement-a-chaud.md`.

---

## Contexte

Le déploiement à chaud — faire évoluer une instance en service, sans interruption — est un
objectif à forte valeur. L'analyse montre que la plupart de ses fondations sont **déjà
acquises** par des décisions antérieures prises pour d'autres raisons : c'est la reconnaissance
d'une convergence, plus quelques décisions neuves, pas une fonctionnalité à bâtir de zéro.

Ce que le déploiement déplace n'est pas « une colonne » mais **la version d'un niveau**
(Fabrica ou projet), avec tout ce qu'elle emporte (schéma, rôles, ACL, catalogues,
fonctionnalités, règles). Il opère sur une **instance** (plan d'exécution, ADR-0013).

## Définitions

- **Instance** : un ou plusieurs serveurs applicatifs connectés à une **même base**. Si cette
  base est elle-même répartie, c'est un **cluster** vu par les serveurs applicatifs comme une
  base unique. Une instance est un déploiement concret (dev, recette, prod) d'un couple de
  versions (Fabrica, projet).
- **Niveaux déployables** : Fabrica et projet (les deux plans définitionnels de l'ADR-0013).
- **Montée** : passage d'une instance d'un couple de versions à un autre.

## Décision — mode séquentiel

La montée est **séquentielle**, jamais atomique-simultanée sur les deux niveaux :

- **Montée d'un seul niveau** : Fabrica seule, ou projet seul. Garantie par la compatibilité
  ascendante (projet_v1 tourne sur Fabrica_v2) — c'est ce qui protège le client d'être forcé
  de tout refaire à chaque montée de Fabrica.
- **Montée conjointe (cas fréquent)** : Fabrica et projet montent ensemble (le projet veut une
  capacité nouvelle de Fabrica, ou un besoin métier traverse les deux niveaux). Elle est
  **séquencée** : on monte Fabrica d'abord — l'instance tourne alors en Fabrica_v2 + projet_v1,
  état **garanti compatible** — puis on monte le projet. On s'appuie sur la compatibilité
  ascendante précisément pour rendre cet état intermédiaire sûr.

> Distinction à ne jamais confondre : l'indépendance des niveaux est une propriété de
> **compatibilité** (projet_v1 marche sur Fabrica_v2), pas une prédiction de **fréquence** (en
> pratique les deux montent souvent ensemble). Les deux sont vraies, à des niveaux différents.

## Décision — invariant « au plus deux versions actives »

À tout instant, au plus **deux versions** coexistent sur une instance (l'ancienne et la
nouvelle). Conséquences :

- **Rétro-compatibilité bornée** : Fabrica_vN ne doit jamais lire que le format **courant et le
  précédent** — jamais tout l'historique. C'est ce qui empêche la dette de rétro-compatibilité
  d'exploser (sans cet invariant, vN devrait lire v1, v2, v3…).
- **Montées sérialisées** : on ne démarre pas une nouvelle montée tant que la précédente n'a
  pas fait son ménage (sinon trois versions coexisteraient — violation). Une à la fois par
  instance, ménage compris.
- **Grosse évolution = plusieurs montées successives** : v1→v2 (ménage), v2→v3 (ménage)… jamais
  un saut direct v1→v5.

## Décision — cycle d'une montée (expand → ménage)

Une montée se déroule en gestes, selon le motif expand/contract (Principe VI) étendu de « le
schéma » à « la version complète d'un niveau » :

1. **Expand** : poser le nouveau (nouveau format/colonne/rôle/schéma) **à côté** de l'ancien ;
   le code de la nouvelle version **écrit le nouveau et lit les deux**.
2. **Coexistence** : les deux versions actives cohabitent ; l'instance fonctionne.
3. **Validation** : tous les gestes de la montée sont accomplis, plus rien ne dépend de
   l'ancienne version.
4. **Ménage (contract)** : retrait de l'ancienne ; la nouvelle version devient pleinement
   active **et seule**.

Le déclencheur du ménage n'est pas un délai vague mais **la complétion validée de la montée** ;
l'invariant des deux versions rend non ambigu ce que « l'ancienne » désigne. Le ménage n'est
jamais dans la même livraison que l'expand.

## Décision — évolutions internes de Fabrica

Le même motif s'applique aux évolutions de la **mécanique interne** de Fabrica (p. ex. changer
la façon dont un paramétrage est stocké), pas seulement au métamodèle métier : expand (poser le
nouveau format, lire les deux), coexistence, ménage. Le code de Fabrica_vN **porte une capacité
de rétro-lecture** du format précédent pendant la fenêtre de coexistence — bornée à une version
en arrière par l'invariant. C'est un coût récurrent assumé de l'interne à chaud.

## Décision — déploiement multi-serveurs applicatifs (rolling update)

Une instance pouvant avoir N serveurs applicatifs devant la base unique, déployer à chaud =
**rolling update** : mettre à jour les serveurs un par un pendant que les autres servent.
Pendant la transition, des serveurs v1 et v2 coexistent devant la même base — possible **parce
que** l'état intermédiaire est compatible (additivité, invariant des deux versions). C'est le
**même mécanisme** que la coexistence des versions, vu sous l'angle des nœuds : aucun mécanisme
neuf.

## Décision — prérequis de base (hypothèse fondatrice)

**Fabrica exige une base logiquement unique présentant la sémantique relationnelle cohérente et
transactionnelle de PostgreSQL** : cohérence forte, transactions ACID, intégrité référentielle.

- La **réalisation** (serveur unique ou cluster présentant une vue de base unique et cohérente)
  est à la charge de l'infrastructure et **hors périmètre** de Fabrica.
- Le critère est la **propriété**, pas l'étiquette : toute base ne fournissant pas cette
  sémantique — typiquement les bases à **cohérence éventuelle** — est hors périmètre. Une base
  distribuée qui honore la sémantique relationnelle cohérente (type NewSQL) n'est pas exclue
  par principe.
- Ce prérequis **généralise l'ADR-0004** : le choix de PostgreSQL n'était pas une préférence
  mais l'exigence de cette sémantique, dont dépendent l'orchestration d'écriture atomique
  (ADR-0005), la propagation d'identité en transaction (ADR-0010), la RLS/intégrité. Une base
  à cohérence éventuelle les ferait toutes s'effondrer.

## Ce qui est acquis / neuf

**Acquis (déjà décidé, révélé comme du à-chaud) :**
- schéma de base à chaud → expand/contract append-only (ADR-0002) ;
- non-corruption pendant l'évolution → migrations transactionnelles (Principe VI) ;
- schéma GraphQL à chaud → **nativement** : un client n'interroge que ce qu'il connaît ; ajouter
  types/champs est transparent ; seule la suppression d'un champ encore utilisé casse — cas
  exceptionnel et planifié (le contract) ;
- coexistence sûre de deux versions → additivité générale (rôles/ACL ADR-0012, points
  d'extension ADR-0013).

**Neuf (décisions de cet ADR) :** mode séquentiel, invariant des deux versions + sérialisation,
cycle expand→ménage explicite, rétro-lecture bornée pour l'interne, rolling update, prérequis de
base.

## Conséquences

- **Discipline permanente** : toute évolution (métamodèle, interne, version de niveau) doit être
  pensée « ancienne et nouvelle doivent coexister le temps de la montée ». Ce n'est pas une
  fonctionnalité codée une fois, c'est une contrainte sur chaque changement futur — la même
  discipline expand/contract déjà acceptée pour la base, étendue à tout.
- **Traçage des versions de format dans l'instance** : le « ménage » a besoin de savoir qu'un
  format n'est plus utilisé. L'instance porte la trace des versions de format présentes. Mécanisme
  renvoyé à l'ADR « données système / montées de version ».
- **Haute disponibilité** graduée (hors périmètre de démarrage) : base unique d'abord ; failover
  passif (standby) si la disponibilité l'exige ; multi-actif seulement sur besoin réel avéré,
  qui rouvrirait les ADR concernés.