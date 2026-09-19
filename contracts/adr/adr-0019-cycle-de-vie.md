# ADR-0019 — Cycle de vie des objets : états et transitions

**Statut :** Proposé — 2026-07-11
**Portée :** décision du **cœur** (Fabrica), générique (Principe IV). Mécanisme de premier rang,
optionnel par entité. S'appuie sur les listes de valeurs (ADR-0018), les effets (ADR-0006/0016),
l'autorisation (ADR-0012), l'i18n (ADR-0017). À éclater en `contracts/adr/0019-cycle-de-vie.md`.

---

## Contexte

Un objet peut avoir un **cycle de vie** : un **état** (nouveau, planifié, clos…) et des **transitions
autorisées** entre états. C'est un mécanisme de premier rang, mais **optionnel** (toutes les entités
n'ont pas de cycle de vie). Fabrica fournit le *mécanisme* ; les états et transitions, étant métier,
sont **définis par le projet** — Fabrica **n'impose aucun état** (ils dépendent de l'entité et de ses
usages ; imposer serait une contamination domaine → cœur, Principe IV).

## Décision — activation par entité

- Une entité est **marquée « à états »**, ce qui active les mécanismes du cycle de vie.
- **Déclarer un champ `state` à la main est interdit** (message explicatif indiquant la bonne façon :
  marquer l'entité « à états »). L'état est un attribut standard *sous le capot*, piloté par une
  déclaration de haut niveau, jamais bricolé.

## Décision — l'état est une liste de valeurs

Les états possibles sont une **liste de valeurs** (ADR-0018) : code, libellé (traduit, ADR-0017),
ordre, actif/inactif. Les états bénéficient donc de tout le mécanisme des listes (édition des
libellés, i18n, désactivation d'un état obsolète).

**Nommage — un état est un qualificatif, jamais un verbe d'action.** « planifié » est un état ;
« planifier » est une action (une transition). Cette règle maintient la grammaire du système : les
états décrivent ce qu'un objet **est** (valeur stable), les transitions ce qu'on lui **fait**. La
convention porte sur le **code** et le **libellé** ; au-delà (traductions, spécificités projet), elle
reste **documentaire** — le projet fait ce qu'il veut.

## Décision — le graphe de transitions

- Le graphe des transitions autorisées (« de l'état X on peut aller vers Y ou Z ») est **déclaré dans
  le métamodèle** (partie de la définition du projet, source de vérité dans git). Il se change par le
  **cycle de déploiement, avec tests** — **pas éditable à chaud** (un cycle de vie mal modifié casse
  le comportement métier de façon subtile ; le figer force le passage par les tests, Principe II).
- Comme tout le métamodèle, il est **projeté en table en lecture seule** pour être **consultable et
  réflexif en RUN** (un objet peut savoir quelles transitions lui sont ouvertes ; un écran, un script
  peuvent interroger le cycle de vie). La table est un **reflet** de la source git, régénéré au
  déploiement — jamais une source modifiable (sinon divergence, et métamodèle hors git/hors tests).
- Distinction : les **valeurs** d'état sont une liste de valeurs (ADR-0018) ; les **transitions
  autorisées** entre elles sont de la **structure métamodèle**, pas une liste de valeurs.

## Décision — transitions insurpassables (règle d'intégrité)

Une transition hors du graphe est **interdite à quiconque, administrateur inclus** — c'est une
**règle d'intégrité, pas une autorisation**. Raison : des **systèmes connectés et plugins** (présents
et à venir) s'appuient sur le fait qu'un objet est **toujours dans un état légal du graphe**. Un objet
hors-graphe romprait ce contrat et casserait la logique des consommateurs du modèle. **Aucun forçage**,
même admin. Un objet réellement bloqué se débloque en **corrigeant le graphe et en redéployant** (avec
tests), jamais en forçant un état illégal. (Cohérent intégrité vs autorisation, ADR-0006.)

## Décision — la validation n'est pas un mécanisme dédié

« Une transition soumise à validation » ne se modélise **jamais** comme un mécanisme d'approbation
spécial. Deux cas, chacun ramené à l'existant :
1. **Valideur différent de l'acteur** → un **état intermédiaire** (« en attente de validation ») dans
   le graphe ; le passage au suivant est une transition normale faite par le valideur. Aucun
   mécanisme neuf — un état de plus.
2. **Même personne valide avant de transiter** → un **attribut rendu obligatoire** à la transition,
   qui est la **justification** du changement d'état (son nom dépend du cas : compte-rendu de
   traitement, analyse technique…). Réutilise `rendre_obligatoire` (ADR-0006). Fabrica fournit le
   mécanisme ; le projet nomme le champ.

## Décision — greffes sur les mécanismes existants

Une transition peut déclencher, **sans mécanisme neuf** :
- une **cascade** (« clore la tâche ferme le ticket ») → `apresModif`, moteur des effets (ADR-0016) ;
- une **autorisation** (« seul un manager valide ») → ADR-0012 ;
- des **effets liés à l'état, réversibles** (date figée en « planifié », re-modifiable en « à
  planifier ») → tombe du **point fixe** : les effets sont **recalculés selon l'état courant**, jamais
  cumulés dans le temps (ADR-0016).

## Décision — déclenchement des transitions

- **Manuel ou via API** par défaut.
- **Automatique réservé** (transitions temporelles ou par héritage) — non construit au démarrage.

## Hors périmètre (renvois / dettes)

- **Affectation / responsabilité** (une transition affecte l'objet à une personne/groupe) : concept
  non défini, plus large que le cycle de vie (modèle d'assignation, conséquences, file de travail).
  **ADR distinct, version ultérieure de Fabrica.**
- **Liens inter-entités** (« toutes les tâches traitées → la demande traitée ») : extension de la
  cascade au cas inter-entités ; mécanisme et coût (remontée vers le parent) à préciser. À traiter
  (peut être avec l'affectation ou à part).
- **Workflow avancé** : transitions parallèles, approbations multi-acteurs, SLA, temporel. Réservé.
- **Métamodèle réflexif en table (lecture seule)** : principe général dégagé ici mais valant pour
  **tout** le métamodèle (entités, attributs, listes…). ADR « réflexivité du métamodèle » à poser.
- **Saisie / source de vérité du métamodèle** : réconcilier l'édition tabulaire efficace (quatre
  tables : objets, attributs, listes, valeurs de listes) avec le versionnement git. ADR distinct.

## Alternatives écartées

- **Champ `state` déclaré à la main** : bricolage hors mécanisme. Remplacé par « entité à états ».
- **Graphe de transitions éditable à chaud** : un cycle de vie mal modifié casse le métier ; doit
  passer par les tests. Figé dans le métamodèle, changé par déploiement.
- **Forçage admin d'une transition** : romprait le contrat d'intégrité sur lequel s'appuient les
  systèmes connectés. Insurpassable par tous.
- **Mécanisme d'approbation dédié** : inutile — une validation est soit un état, soit un attribut
  obligatoire.
- **États standards imposés par Fabrica** : les états sont métier (dépendent de l'entité). Le cœur
  n'en impose aucun.
