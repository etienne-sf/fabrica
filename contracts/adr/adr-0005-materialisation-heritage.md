# ADR-0005 — Matérialisation de l'héritage : table par classe avec ligne à chaque niveau

**Statut :** Proposé — 2026-07-11
**Portée :** décision du **cœur** (Fabrica), formulée en termes génériques. L'héritage est un
mécanisme de modélisation offert à tout métamodèle-client ; le cœur en fournit la
matérialisation sans connaître aucun domaine applicatif (Principe IV). Les exemples concrets
d'un domaine particulier sont relégués à la section « Illustration », explicitement hors du
contrat du cœur. À éclater en `contracts/adr/0005-materialisation-heritage.md`.

---

## Contexte

Un métamodèle-client peut déclarer des **relations d'héritage** entre entités (une classe
« étend » une autre). Le cœur doit générer, à partir de cette déclaration, une matérialisation
relationnelle (tables PostgreSQL). Trois **exigences génériques** — qu'un métamodèle-client
est susceptible d'imposer — contraignent ce choix :

1. **Instanciation à tout niveau.** Une entité peut être créée à un niveau *intermédiaire* de
   la hiérarchie, sans être spécialisée. Une classe peut donc être **concrète et non-feuille**
   à la fois : instanciable directement, tout en ayant des sous-classes.
2. **Spécialisation progressive.** Un enregistrement créé à un niveau peut recevoir une
   spécialisation ultérieure (gagner un niveau plus dérivé) **sans changer d'identité** et sans
   être détruit puis recréé.
3. **Requête polymorphe sur une classe mère.** Interroger une classe mère, toutes sous-classes
   confondues, est un besoin courant (recherches, rapports), potentiellement sur de gros
   volumes.

Le cœur ne présume pas que ces exigences sont *toujours* activées ; il fournit une
matérialisation qui les honore quand un métamodèle-client les mobilise. (Distinction hors
périmètre : les **colonnes de base techniques** — identifiant, horodatage… — ajoutées
systématiquement à toute table ne relèvent pas de l'héritage ; c'est un socle uniforme, traité
ailleurs.)

## Décision

Le cœur matérialise l'héritage selon le motif **table par classe avec une ligne à chaque
niveau** (« Class-Table Inheritance ») :

- **Chaque classe de la hiérarchie a sa propre table**, y compris les classes intermédiaires.
- **Un enregistrement existe comme une ligne à chaque niveau** dont il relève, ces lignes
  partageant le **même identifiant**. Un enregistrement d'une sous-classe possède une ligne au
  niveau de cette sous-classe *et* une ligne à chacun des niveaux ancêtres ; un enregistrement
  instancié à un niveau intermédiaire ne possède de ligne que jusqu'à ce niveau.
- **La création est atomique et portée par le cœur** : instancier une classe insère, dans une
  seule transaction, une ligne à chaque niveau de la chaîne, de la racine jusqu'au niveau visé.
  La logique de propagation n'est **jamais** laissée à l'appelant.
- **La spécialisation progressive** se fait en *ajoutant* une ligne au niveau plus dérivé, avec
  le même identifiant, sans détruire/recréer l'enregistrement ni migrer ses données.
- **La suppression** est propagée symétriquement : retirer, atomiquement, les lignes de
  l'enregistrement à tous ses niveaux.
- **La requête polymorphe** sur une classe mère interroge directement la table de ce niveau,
  qui contient l'ensemble des enregistrements de ses sous-classes — sans vue ni `UNION`.

## Raison

- C'est la **seule** des trois stratégies standard qui satisfait les trois exigences
  génériques *simultanément* : instanciation à tout niveau (chaque niveau a sa table),
  spécialisation progressive (ajout d'une ligne au niveau inférieur), requête polymorphe
  efficace (la table mère existe et contient tous ses descendants).
- Le coût principal — propagation à l'écriture — est **cohérent avec le profil visé** d'un
  outil de référentiel : massivement lu, peu écrit. Payer à l'écriture pour gagner en lecture,
  en justesse de modélisation et en interrogeabilité polymorphe est le bon échange.
- Le cœur reste **ignorant du domaine** : il implémente un mécanisme d'héritage générique et
  n'impose aucune restriction sur *quoi* hérite de *quoi* — laissé libre au métamodèle-client.

## Alternatives écartées

- **Table par classe concrète (feuilles seules), mère dissoute** (« Table-Per-Concrete-
  Class »). Une table par feuille, colonnes de la mère recopiées dans chaque définition, pas de
  table pour les classes mères. *Rejetée* : elle suppose que seules les feuilles sont
  instanciables et n'a donc **nulle part où stocker un enregistrement créé à un niveau
  intermédiaire** (exigence 1 non satisfaite). Elle ne permet pas non plus la spécialisation
  progressive (exigence 2). Enfin, la requête polymorphe (exigence 3) y impose un `UNION` sur
  toutes les feuilles, coûteux quand les sous-classes sont nombreuses.
- **Variante « feuilles + vues d'agrégation »**. Matérialiser les feuilles comme ci-dessus et
  reconstituer les classes mères par des vues qui `UNION`-nent les filles. *Rejetée* : les vues
  d'union ne sont pas des cibles d'écriture fiables (ni `INSERT`, ni `UPDATE` à travers
  l'union), et surtout elle hérite du défaut précédent — aucun emplacement pour instancier un
  niveau intermédiaire (exigences 1 et 2 non satisfaites).
- **Table unique par hiérarchie** (« Single-Table Inheritance »). Une seule table pour toute la
  branche, colonne de type, colonnes de toutes les sous-classes réunies (majoritairement
  nulles). Lecture, écriture et requête polymorphe triviales. *Rejetée* : sur une hiérarchie
  large, elle produit une **explosion du nombre de colonnes** (table ingérable), et une
  prolifération de colonnes nullables sans garantie structurelle.

## Conséquences

- **La propagation à l'écriture est de la logique de mutation générée par le cœur, et doit être
  atomique.** Un chemin d'écriture qui oublierait un niveau produirait un enregistrement
  incohérent (présent au niveau spécialisé, absent à un niveau ancêtre). Cette orchestration
  multi-tables est un **critère de premier plan pour le choix du moteur GraphQL** (ADR à
  venir) : les moteurs auto-générés portent le moins naturellement l'écriture custom
  multi-tables — à peser explicitement.
- **L'identité partagée entre niveaux est l'invariant critique.** Même identifiant à chaque
  niveau, cohérence garantie (identité générée au niveau racine, clés étrangères entre
  niveaux). Cet invariant DOIT figurer dans `tests/acceptance/` (gelé) : une désynchronisation
  d'identité entre niveaux effondre le modèle silencieusement.
- **La lecture d'un objet complet exige de joindre les niveaux** : pendant du coût d'écriture,
  relève de la stratégie anti-N+1 / pushdown côté moteur et résolveurs.
- **L'évolution du métamodèle** (ajout/retrait d'un niveau ou d'un attribut) reste gouvernée
  par l'expand/contract et les migrations (Principe VI) : cet ADR fournit la structure cible,
  il ne modifie pas ce régime.
- **Coût selon la forme de la hiérarchie** : ce motif ne souffre pas de l'explosion de
  colonnes ; le coût croît avec la **profondeur** (nombre de niveaux à insérer/joindre par
  enregistrement), pas avec la largeur. Une garde de sanité sur la profondeur pourra être
  ajoutée si nécessaire (non bloquant).

## Illustration par un domaine client (hors contrat du cœur)

> Cette section est **illustrative et ne fait pas partie du contrat du cœur**. Elle donne des
> contre-exemples concrets, issus du premier domaine-client (référentiel d'architecture
> d'entreprise et inventaire de configuration associé), qui **réfutent les alternatives
> écartées**. Ces concepts n'entrent pas dans le cœur ; ils motivent seulement les exigences
> génériques ci-dessus.

- **Contre-exemple à « feuilles seules » (exigences 1 et 2).** On crée un objet à un niveau
  intermédiaire sans le spécialiser — typiquement un élément déclaré à un niveau générique
  avant que sa nature précise soit connue — puis on le spécialise plus tard. Une stratégie qui
  ne matérialise que les feuilles n'aurait aucune table pour l'accueillir à sa création, ni
  moyen de le spécialiser sans le recréer.
- **Contre-exemple à « table unique » (explosion de colonnes).** Une classe racine très large,
  mutualisant les attributs de très nombreuses sous-classes, atteint en pratique plusieurs
  centaines de colonnes dans une approche mono-table (cas observé : plus de 250 colonnes),
  rendant la table ingérable.
- **Besoin de requête polymorphe (exigence 3).** Les recherches et rapports transverses
  interrogent régulièrement une classe mère abstraite tous types confondus, sur de gros
  volumes, ce qui justifie que la table de la classe mère existe et soit directement
  interrogeable.
