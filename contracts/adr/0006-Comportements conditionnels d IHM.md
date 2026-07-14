# ADR-0006 — Comportements conditionnels d'IHM : catalogue fermé d'effets déclaratifs

**Statut :** Proposé — 2026-07-11
**Portée :** décision du **cœur** (Fabrica), formulée en termes génériques (Principe IV).
Concerne les comportements d'interface conditionnels d'un formulaire. À éclater en
`contracts/adr/0006-effets-ihm-declaratifs.md`.

> Numérotation indicative : deux ADR discutés antérieurement restent à rédiger (choix du
> moteur GraphQL ; isolation des projets-clients vis-à-vis du moteur). La numérotation
> définitive sera réconciliée à leur écriture.

---

## Contexte

Un formulaire a besoin de comportements conditionnels : selon la valeur d'un champ, un autre
champ est grisé, masqué, vidé, rendu obligatoire, etc. La tentation initiale était de
concevoir un modèle général de « règles » exécutant du code custom (avec sandbox, choix de
langage, etc.). Cette voie a été **écartée pour le démarrage** : concevoir un modèle de
règles complet à partir de quelques exemples serait de la sur-conception spéculative
(contraire au Principe VII), et de nombreux cas non encore rencontrés pourraient invalider le
modèle choisi trop tôt.

Distinction fondatrice retenue : une même **intention métier** peut se manifester à deux
couches — comme **effet d'interface** (confort utilisateur, au navigateur) et comme **règle
de donnée** (intégrité, autorité serveur). Ces deux manifestations partagent une *condition*
mais produisent des *effets de nature différente* (« griser un champ » n'est pas « asserter
qu'un champ est vide »). Le présent ADR ne traite que des **effets d'interface**. Les règles
de donnée relèvent du mécanisme de validation (autorité serveur), séparément.

## Décision

Fabrica offre un **catalogue fermé d'effets d'interface déclaratifs**. Les projets ne
déclarent pas de code : ils référencent des effets connus de Fabrica, conditionnés par des
tests simples sur des champs.

**Structure d'une règle d'interface :**
- une **condition** : un test simple portant sur un ou des champs du formulaire courant ;
- une **liste d'actions**, chaque action étant un couple **(effet, cible(s))**. Une condition
  peut donc déclencher plusieurs effets, sur plusieurs cibles, y compris plusieurs effets sur
  une même cible.

Forme (illustrative) :
```
quand <condition sur champ(s)> :
  - griser   [champ_A, champ_B]
  - masquer  [champ_C]
  - vider    [champ_A]
```

**Catalogue d'effets initial (fermé) :**
- `griser` (lecture seule)
- `masquer`
- `vider`
- `rendre_obligatoire`
- `rendre_optionnel`

**Nature et autorité :**
- Un effet d'interface est **projeté vers le navigateur uniquement** ; le rendu l'applique.
- Un effet d'interface est **du confort, jamais une autorité**. Il ne garantit aucune
  intégrité de donnée. L'intégrité correspondant à la même intention métier est portée,
  séparément, par une **règle de donnée** (validation, autorité serveur), qui seule fait foi.
  Le navigateur peut être contourné ou absent ; la garantie vient toujours de la couche
  donnée.
- La **condition** est une expression pure (aucun accès au monde) : elle est ainsi évaluable
  au navigateur. Si une condition devait dépendre de données que seul le serveur possède,
  elle cesserait d'être évaluable côté client seul — cas à déclarer, non couvert au démarrage.

**Implémentation portée par Fabrica :** pour chaque effet du catalogue, Fabrica écrit **une
seule fois** le comportement de rendu (côté navigateur) et la validation des incompatibilités.
Les projets ne référencent que le *nom* de l'effet. Ils n'écrivent aucun code d'effet, ne
dépendent ni du moteur GraphQL ni d'un runtime : un comportement conditionnel d'IHM reste donc
**entièrement déclaratif et portable**.

## Gouvernance du catalogue

- **Pas de seuil d'ouverture chiffré.** À chaque nouvelle règle qui ne rentre pas dans le
  catalogue, on se pose explicitement la question — jamais par compteur automatique.
- **Critère d'admission au catalogue** (le garde-fou qui remplace le seuil) : un effet n'entre
  au catalogue que s'il est **générique, réutilisable, et déclarable en une ligne lisible**.
  Un effet spécifique à un cas particulier NE DOIT PAS être ajouté au catalogue : son
  apparition est le **signal** qu'on atteint la limite du déclaratif, et qu'il faut envisager
  l'ouverture du runtime de règles (ci-dessous) plutôt que de faire fuir le catalogue vers un
  fourre-tout, une exception à la fois.
- **Incompatibilités entre effets sur une même cible :** le catalogue déclare les
  combinaisons interdites, que Fabrica refuse à la validation du métamodèle. Exemples de
  contradictions à rejeter : `rendre_obligatoire` + `rendre_optionnel` sur la même cible ;
  `rendre_obligatoire` + `masquer` (champ requis mais invisible) ; `rendre_obligatoire` +
  `griser` ou + `vider` (champ requis mais non saisissable / vidé). La liste exacte est un
  détail d'implémentation à fixer.

## Réservé mais NON construit

Le **runtime de règles custom** (bac à sable JS hébergé par Fabrica, exécution isomorphe
serveur/navigateur pour les règles pures) reste la **porte de sortie** pour le jour où le
catalogue fermé ne suffit plus. Il n'est **pas** construit au démarrage. Son ouverture se
décide au cas par cas, déclenchée par le critère d'admission ci-dessus (un besoin qu'aucun
effet générique ne couvre), jamais par un seuil. Conséquence importante : **au démarrage,
aucun choix de langage/sandbox n'est requis** — cette question est repoussée avec le runtime.

## Hors périmètre (à ne pas confondre)

- **Filtrage d'une liste fille selon une liste mère.** Ce n'est **pas** un effet d'interface.
  C'est une **contrainte relationnelle structurelle**, traitée ailleurs (table de couples
  valides + clé étrangère composite, cf. décisions d'intégrité). Côté donnée, la FK refuse les
  couples invalides ; côté IHM, la liste fille se *peuple* par une **requête** sur cette table
  filtrée — ce n'est pas un « effet » appliqué mais une lecture de donnée de référence. Ne pas
  l'implémenter comme une règle d'interface.
- **Conflits entre règles sur une même cible** (deux règles donnant des verdicts opposés sur
  le même champ ; ordre, priorité, cascade). **Hors périmètre au démarrage.** Le démarrage
  suppose des règles **indépendantes** (pas deux règles visant la même cible). Une règle qui
  vise une cible déjà visée par une autre est un **drapeau** : elle relève de la question à se
  poser (cf. gouvernance), pas d'un traitement automatique.
- **Conditions composées** (ET/OU de plusieurs tests, dépendance à un état serveur). Au
  démarrage, les conditions sont des **tests simples** sur des champs du formulaire. Une
  condition qui se complexifie est un signal côté *condition* (et non côté effet) : elle
  relève elle aussi de la question à se poser.

## Limites et obligation de test

Ce mécanisme est un accélérateur déclaratif, **pas une garantie de correction**. Il n'est
**pas possible de sécuriser à 100 % le résultat produit** par la combinaison des conditions et
des effets, et ce pour des raisons structurelles, indépendamment de la qualité de
l'implémentation :

- au démarrage, plusieurs cas sont explicitement non couverts (conflits entre règles sur une
  même cible, conditions composées) — leur résultat n'est pas défini ;
- même dans le périmètre couvert, une règle peut être **correctement exécutée mais mal
  conçue** : une condition qui ne capture pas l'intention réelle, un effet appliqué à la
  mauvaise cible, une combinaison d'effets dont l'interaction n'avait pas été anticipée ;
- la validation des incompatibilités attrape les contradictions *déclarées*, pas les
  comportements simplement indésirables.

**Conséquence, conforme au Principe II :** la correction du comportement produit par une règle
est établie par des **tests**, jamais par l'existence du mécanisme. Chaque règle configurée
dans un métamodèle doit être **testée pour son résultat effectif** (« quand la condition est
vraie, la cible est bien dans l'état attendu ; quand elle est fausse, elle ne l'est pas »).
Ce n'est pas une surprise — c'est le rappel que le déclaratif réduit le code à écrire, mais ne
dispense pas de vérifier ce qu'il produit. En particulier, comme un effet d'IHM n'a **aucune
autorité**, un test qui ne vérifierait que le comportement d'écran ne prouve rien sur
l'intégrité : celle-ci est prouvée par les tests de la **règle de donnée** correspondante,
côté serveur.

## Alternatives écartées

- **Runtime de règles custom dès le départ** (code JS/autre en sandbox). *Écartée* comme
  prématurée : concevoir le modèle de règles sur quelques exemples risque de le figer faux, et
  impose de trancher tout de suite langage, sandbox, contexte d'exécution — complexité non
  justifiée tant que le déclaratif suffit.
- **« Une règle, deux effets » comme modèle général** (une déclaration produisant
  mécaniquement l'effet-donnée et l'effet-IHM). *Écartée pour le démarrage* : la traduction
  d'une intention vers un comportement d'écran n'est pas mécanique (« doit être vide » ne
  désigne pas « griser » plutôt que « masquer » ou « vider »). Le catalogue fermé, plus
  modeste, évite de présumer cette traduction. Une consolidation ultérieure (condition
  partagée alimentant à la fois l'effet d'IHM et la règle de donnée) reste possible, non
  construite.

## Conséquences

- Le métamodèle gagne, par vue, une liste de règles d'interface (condition + actions). **Garde
  de sanité :** chaque action doit rester déclarable en une ligne lisible ; le jour où une
  action demande un paragraphe, c'est qu'elle n'a pas sa place dans un catalogue déclaratif.
- **Le résultat des règles doit être testé** (cf. « Limites et obligation de test ») : le
  mécanisme ne garantit pas la correction, les tests l'établissent.
- **Portabilité maximale pour cette classe de règles** : aucun code de projet, donc aucune
  dépendance au moteur GraphQL ni à un runtime. Un changement de moteur n'impacte pas ces
  comportements. Cela répond, pour ce cas, à l'exigence de pérennité des projets-clients.
- **Aucun choix de langage/sandbox requis au démarrage** (repoussé avec le runtime).
- Fabrica porte l'implémentation de chaque effet (rendu navigateur + validation des
  incompatibilités) une fois pour toutes ; les projets ne font que référencer des effets par
  nom.
- L'intégrité de donnée correspondant à une intention métier reste portée par une **règle de
  donnée** distincte (autorité serveur) ; l'effet d'IHM ne la garantit jamais.

## Illustration (hors contrat du cœur)

> Exemples concrets, non normatifs, pour lecture seule.
> - *Effet simple :* « quand la case `sans_impact` est cochée, `griser` le champ
>   `description_impact` ». L'intégrité correspondante (« si `sans_impact` alors
>   `description_impact` doit être vide ») est une règle de donnée séparée, côté serveur, qui
>   seule fait foi.
> - *Effets multiples :* une même condition peut `griser` plusieurs champs, ou `griser` **et**
>   `masquer` le même champ.