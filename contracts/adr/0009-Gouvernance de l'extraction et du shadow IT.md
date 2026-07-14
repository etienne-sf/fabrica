# ADR-0009 — Gouvernance de l'extraction et du shadow IT (différé)

**Statut :** Différé — 2026-07-11
**Portée :** décision du **cœur** (Fabrica), générique (Principe IV). Configurable par
criticité du projet-client. À éclater en `contracts/adr/0009-gouvernance-extraction.md`.

> **Numérotation indicative.** Cet ADR suppose l'existence d'un ADR « L'API est la frontière
> de sécurité ; le front-end ne fournit aucune sécurité » (issu de la même discussion, à
> rédiger — pressenti ADR-0008). Le présent ADR en est le prolongement côté *usage aval*.
> Numérotation à réconcilier.

---

## Contexte

Fabrica expose une API (le périmètre fonctionnel d'un projet = son métamodèle) à des
consommateurs, y compris des applications externes légitimes. Deux risques d'usage aval ont
été identifiés :

1. **Référentiel parallèle (shadow IT).** Un utilisateur ou un service aspire les données via
   l'API, les stocke ailleurs, y ajoute ses propres règles de gestion, et finit par détenir
   une version « plus à jour » que le référentiel officiel. Résultat : il n'y a plus de
   référentiel — la vérité s'est déplacée hors de l'outil.
2. **Confidentialité de la donnée critique.** Une fois des données sorties de l'application,
   on ne maîtrise plus ce qu'il en advient ; le niveau de confidentialité demandé n'est plus
   garanti.

Poids variable selon le projet-client : **faible pour un référentiel EA**, **fort et
probable pour une CMDB / de l'ITSM** avec de la donnée critique.

## Vérités structurantes (à ne pas oublier)

- **On ne peut pas empêcher techniquement l'extraction de ce qu'un humain a le droit de
  voir.** Toute donnée affichée est copiable (recopie, capture, photo de l'écran). Le « droit
  de consulter sans droit d'extraire » n'est **pas** une garantie techniquement atteignable
  pour un lecteur humain autorisé.
- **Conséquence :** la confidentialité de la donnée critique se joue **à l'accès** (qui a le
  droit de voir quoi — RLS, serveur + base), **pas à la sortie**. Pour la donnée réellement
  sensible, la bonne réponse est parfois « cet utilisateur ne doit pas la voir du tout »,
  plutôt que « il la voit mais ne l'extrait pas ».
- **Ce qui est combattable**, en revanche, c'est l'extraction **massive et programmatique**
  (celle qui alimente un référentiel parallèle) — distincte de la copie humaine anecdotique.
  On ne l'empêche pas absolument ; on la rend visible, attribuable, bornée.

## Décision

**Le moteur de détection de comportements déviants (analyse des requêtes à la volée) est
reporté** : il n'est pas construit au démarrage (risque faible sur le premier projet-client,
EA ; conforme au Principe VII — ne pas construire pour un besoin encore absent).

**MAIS deux prérequis de cette gouvernance ne sont PAS reportables et sont calés dès le
départ** (Principe VII — caler la frontière, pas la politique) :

1. **Journalisation d'accès (audit).** Qui lit/écrit quoi, en quel volume, quand. Sans
   historique constitué *dès le début*, aucune détection ultérieure n'aura de matière à
   analyser. La journalisation est posée tôt, même si son exploitation (détection) est
   différée.
2. **Distinction des identités humain / compte de service.** L'accès humain (via IHM) et
   l'accès programmatique (applications externes) passent par des **identités distinctes**,
   avec scopes propres. Sans cette distinction dès le départ, l'aspiration programmatique
   sera indiscernable de l'usage humain, et toute détection future sera aveugle.

Formulation de synthèse à retenir : **la détection est reportable ; l'audit et la séparation
des identités ne le sont pas.**

## Pistes réservées (pour le jour de l'activation, non construites)

Contre le **référentiel parallèle** (risque 1) :
- accès programmatique **officialisé et scopé** : comptes de service déclarés, limités au
  périmètre fonctionnel du métamodèle ; l'accès légitime est first-class, l'accès détourné
  devient l'anomalie ;
- **quotas de lecture / limitation de débit** : étranglent l'aspiration de masse sans gêner
  la consultation humaine ;
- **détection volumétrique** : un humain consulte quelques fiches, un aspirateur lit des
  dizaines de milliers d'objets — le pattern de volume est le signal ;
- **analyse de requêtes à la volée** pour repérer les comportements déviants (le moteur
  reporté).

Contre la **confidentialité de la donnée critique** (risque 2) :
- **RLS stricte en amont** (seule garantie réelle : besoin d'en connaître) ;
- **marquage / filigrane par utilisateur** et journalisation fine des accès sensibles :
  n'empêche pas la copie, rend l'auteur d'une fuite identifiable (dissuasion + sanction) ;
- **minimisation à l'affichage** : masquer par défaut, révéler à la demande (avec log), ne
  jamais afficher en masse ce qui est critique.

## Configurabilité par criticité du projet

Ces contrôles vivent au **serveur et à la base** (couche Fabrica), **pilotés par la
configuration du projet-client**. Un projet à faible risque (EA) laisse la plupart inertes
(RLS ouverte, audit léger) ; un projet sensible (CMDB) active RLS stricte, quotas, filigrane,
comptes de service obligatoires. Fabrica pose les **emplacements** ; chaque projet règle le
**curseur**.

## Alternatives écartées

- **Empêcher techniquement l'extraction de données visibles** (DRM, « droit de voir sans
  droit d'extraire »). Impossible pour un lecteur humain (analog hole). Rejetée comme
  garantie ; conservée seulement sous forme de *dissuasion* (marquage, log), jamais de
  *garantie*.
- **Authentifier le client / bloquer l'accès programmatique** pour empêcher les applications
  tierces. Impossible (le client est sous contrôle de l'utilisateur) et contre-productif (un
  référentiel a vocation à être interrogé par d'autres systèmes). Remplacée par
  l'officialisation scopée de l'accès programmatique.
- **Tout reporter, y compris audit et identités.** Rejetée : rendrait la détection future
  aveugle et imposerait un rétro-ajout douloureux (refonte de l'identité, absence
  d'historique).

## Conséquences

- Au démarrage : journalisation d'accès **active**, identités humain/service **distinctes**,
  autorisation par attribut (déjà décidée) — c'est tout. Pas de moteur de détection, pas de
  quotas, pas de filigrane tant que le besoin n'est pas réel.
- Le jour d'un projet critique (CMDB) : activation des pistes réservées, sans réécriture des
  fondations, puisque audit et identités sont déjà en place.
- **À retenir absolument :** la confidentialité se garantit à l'accès (RLS), jamais à la
  sortie. Ne pas rebâtir plus tard l'illusion d'un contrôle d'extraction.