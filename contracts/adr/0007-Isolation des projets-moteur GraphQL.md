# ADR-0007 — Isolation des projets-clients vis-à-vis du moteur GraphQL

**Statut :** Proposé — 2026-07-11
**Portée :** décision du **cœur** (Fabrica), générique (Principe IV). Principe d'architecture
dont le choix de moteur (ADR à venir) n'est qu'une application. À éclater en
`contracts/adr/0007-isolation-projets-moteur.md`.

> Numérotation indicative : dans l'ordre logique, cet ADR **précède** l'ADR de choix du moteur
> GraphQL, car il est le principe-parent qui rend ce choix réversible. Numérotation à
> réconcilier.

---

## Contexte

Fabrica est un produit-socle réutilisé par plusieurs projets-clients successifs, développés
par des personnes différentes (cf. constitution, « Contrat public du cœur »). Le moteur
GraphQL retenu (PostGraphile, ADR à venir) est susceptible de changer un jour. Le risque à
neutraliser : qu'un changement de moteur **impacte les projets-clients**, les obligeant à
retoucher leur code — ce qui ruinerait la pérennité du socle.

Le principe qui protège les projets n'est pas de rendre du code *portable entre moteurs*
(faux objectif : du code écrit pour un moteur ne se transpose pas, et bâtir une couche
d'abstraction maison par-dessus le moteur serait de la sur-ingénierie qui fuirait). Le bon
principe est que **les projets ne dépendent jamais du moteur** — s'ils l'ignorent, le changer
ne les touche pas.

## Décision

**Un projet-client ne dépend que du contrat public de Fabrica, jamais du moteur GraphQL.** Le
moteur est un **détail d'implémentation interne** à Fabrica, remplaçable par le fournisseur du
socle sans impact sur les clients. La dépendance est à sens unique : projet → contrat Fabrica
→ (interne) moteur. Un projet ne référence, ne nomme, ni ne présuppose jamais le moteur.

Ce contrat public a **deux surfaces**, et l'isolation doit tenir sur les deux :

**1. Surface d'entrée — la déclaration.** Ce qu'un projet *écrit* (métamodèle : entités,
attributs, contraintes, validations, effets d'IHM du catalogue) est exprimé dans le format
neutre de Fabrica. C'est **Fabrica** qui traduit cette déclaration vers le mécanisme du moteur
courant. Un projet ne écrit jamais une fonction ni un plugin propre au moteur. Cette surface
est neutre par construction.

**2. Surface de sortie — l'API générée.** Ce qu'un projet *consomme* (l'API GraphQL que
Fabrica produit) doit être une **forme définie par Fabrica**, pas la sortie brute du moteur.
C'est le point le plus subtil et le plus facile à rater : un moteur impose ses propres
conventions (nommage, forme des mutations, types de connexion/pagination, identifiants de
nœud, format d'erreurs). Si l'IHM et les consommateurs d'un projet sont écrits contre ces
conventions *brutes*, changer de moteur casse le projet — l'isolation a fui par la sortie,
même si l'entrée était neutre. Donc : **la forme de l'API que les projets consomment fait
partie du contrat public de Fabrica ; les conventions spécifiques au moteur ne doivent pas s'y
manifester.** Selon le moteur, cela suppose soit d'adopter et documenter des conventions
stables imposées par Fabrica, soit une couche de normalisation qui masque les particularités
du moteur (choix d'implémentation, non tranché ici).

## Deux natures de custom (rappel structurant)

- **Custom du cœur** : la logique générique que Fabrica génère et porte (ex. l'orchestration
  d'écriture multi-tables de l'héritage, ADR-0005). Elle est *interne à Fabrica*. Un changement
  de moteur oblige à adapter **Fabrica**, une fois, par le fournisseur du socle — jamais les
  projets.
- **Custom du projet** : les règles propres à un domaine-client. Elles se déclarent **dans le
  métamodèle** (forme neutre) et sont traduites par Fabrica. Un projet n'écrit jamais de code
  lié au moteur.

## État de démarrage : isolation acquise par le tout-déclaratif

Au démarrage, il n'existe **pas de custom de projet exécutable** : un projet ne produit que du
déclaratif (métamodèle, références au catalogue d'effets d'IHM — ADR-0006, runtime de règles
réservé mais non construit). Or le déclaratif ne dépend d'aucun moteur par construction.
**L'isolation, côté entrée, est donc actuellement acquise gratuitement.** Le seul travail réel
d'isolation au démarrage porte sur la **surface de sortie** (empêcher les conventions du moteur
de fuir dans l'API consommée).

## Contrainte pour le futur runtime de règles (non construit)

Le jour où le runtime de règles custom sera ouvert (réservé par l'ADR-0006), il **devra être
hébergé par Fabrica, pas par le moteur** : le code custom d'un projet s'exécutera dans un bac à
sable que Fabrica possède et invoque à sa propre frontière, de sorte que ce code dépende du
*contrat de runtime de Fabrica* et non du moteur. La portabilité vient de *qui héberge
l'exécution*, pas du langage choisi. Cette contrainte est **posée ici comme exigence** que
l'ADR ouvrant le runtime devra satisfaire ; elle n'est pas construite maintenant.

## Alternatives écartées

- **Laisser les projets écrire du code spécifique au moteur** (fonctions/plugins propres au
  moteur). Plus rapide à court terme, mais couple chaque projet au moteur : un changement de
  moteur casse tous les projets. Contraire à la raison d'être du socle. Rejetée.
- **Construire une couche d'abstraction maison « multi-moteurs »** par-dessus le moteur, censée
  rendre le custom portable entre moteurs. Sur-ingénierie qui fuit (les particularités du
  moteur finissent toujours par percer), et faux objectif : la portabilité vient de
  l'hébergement de l'exécution par Fabrica, pas d'une abstraction du moteur. Rejetée.

## Conséquences

- **Fabrica absorbe le risque de changement de moteur, pas les projets.** Un changement de
  moteur = réécriture de la couche de traduction (entrée) et de normalisation (sortie) *dans
  Fabrica*, une fois, par le fournisseur du socle. C'est la valeur même d'un produit-socle :
  concentrer le risque là où le fournisseur peut l'absorber, pour l'ôter des clients.
- **Le choix de moteur devient réversible** (donc peu risqué) : c'est cet ADR qui le rend tel.
  L'ADR de choix du moteur en devient un détail d'implémentation, pas un engagement structurant
  pour les clients.
- **La forme de l'API fait partie du contrat versionné** de Fabrica (cf. constitution,
  compatibilité ascendante) : elle évolue en versions non-cassantes pour les clients,
  indépendamment du moteur sous-jacent.
- **Limite honnête :** l'isolation n'est jamais gratuite sur la surface de sortie. Elle
  n'est aussi solide que la discipline à empêcher les conventions du moteur de fuir dans l'API
  consommée. Une fuite non détectée (un projet qui se met à dépendre d'une particularité
  PostGraphile dans la forme de l'API) réintroduit silencieusement le couplage. Cette
  non-fuite mérite d'être vérifiée (test/revue), pas supposée.
- Le futur runtime de règles héritera de cette contrainte d'hébergement par Fabrica (ci-dessus).

## Parallèle avec le Principe IV

Fabrica pose désormais **deux frontières d'isolation à sens unique**, à ne pas confondre :
- *Principe IV* : le cœur ignore le **domaine** (les métamodèles, dont l'EA, sont des données
  d'entrée).
- *Cet ADR* : les projets ignorent le **moteur** (le moteur est un détail interne de Fabrica).
Le cœur ne connaît pas le domaine ; les projets ne connaissent pas le moteur. Fabrica est
l'intermédiaire qui tient les deux frontières.