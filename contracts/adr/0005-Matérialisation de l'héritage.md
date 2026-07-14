# ADR-0005 — Matérialisation de l'héritage : table par classe avec ligne à chaque niveau

**Statut :** Proposé — 2026-07-11
**Portée :** décision du **cœur** (Fabrica). L'héritage est une notion de modélisation
présente dans tout métamodèle-client ; cette décision est donc générique et ne nomme aucun
concept de domaine (conforme au Principe IV). À éclater en `contracts/adr/0005-materialisation-heritage.md`.

---

## Contexte

Un métamodèle-client peut déclarer des **relations d'héritage** entre entités (« une capacité
EST-UNE brique applicative », « un serveur EST-UN élément de configuration »). Le cœur doit
générer, à partir de cette déclaration, une matérialisation relationnelle (tables PostgreSQL).
Trois faits du domaine, dégagés lors de la conception, contraignent ce choix :

1. **Instanciation à tout niveau.** Une entité peut être créée à un niveau *intermédiaire* de
   la hiérarchie, sans être spécialisée. Exemple : on crée un « serveur » sans savoir encore
   s'il est Windows ou Linux. Une classe peut donc être **concrète et non-feuille** à la fois.
2. **Spécialisation progressive.** Un enregistrement créé à un niveau peut être spécialisé
   plus tard (le serveur devient un serveur Windows), sans changer d'identité.
3. **Requête polymorphe sur les classes mères.** Interroger une classe mère tous types
   confondus (« tous les éléments de configuration », « tous les objets métier ») est un
   besoin réel et fréquent, pour les recherches et les rapports, potentiellement sur de gros
   volumes.

Distinction préalable (hors périmètre de cet ADR mais à ne pas confondre) : les **colonnes de
base techniques** (`id`, `date_maj`…) que le cœur ajoute systématiquement à *toute* table ne
relèvent PAS de l'héritage — c'est un socle technique uniforme, sans table mère ni jointure.
Le présent ADR ne traite que de la matérialisation de l'**héritage fonctionnel** déclaré par
le métamodèle-client.

## Décision

Le cœur matérialise l'héritage selon le motif **table par classe avec une ligne à chaque
niveau** (« Class-Table Inheritance », le modèle qu'emploie ServiceNow) :

- **Chaque classe de la hiérarchie a sa propre table**, y compris les classes intermédiaires.
- **Un enregistrement existe comme une ligne à chaque niveau** dont il relève, ces lignes
  partageant le **même identifiant**. Un serveur Windows = une ligne dans `serveur` + une
  ligne dans `serveur_windows`, liées par identité. Un serveur indéterminé = une ligne dans
  `serveur` uniquement.
- **La création est atomique et portée par le cœur** : instancier une classe insère, dans une
  seule transaction, une ligne à chaque niveau de la chaîne. La logique de propagation n'est
  **jamais** laissée à l'appelant.
- **La spécialisation progressive** se fait en *ajoutant* une ligne au niveau plus dérivé,
  avec le même identifiant, sans détruire/recréer l'enregistrement ni migrer ses données.
- **La requête polymorphe** sur une classe mère interroge directement la table de ce niveau,
  qui contient tous les enregistrements de ses sous-classes — sans vue ni `UNION`.

## Raison

- C'est la **seule** des trois stratégies standard qui satisfait les trois faits du domaine
  simultanément : instanciation à tout niveau (chaque niveau a sa table), spécialisation
  progressive (ajout d'une ligne au niveau inférieur), et requête polymorphe efficace (la
  table mère existe et contient tout le monde).
- Le coût principal — propagation à l'écriture — est **cohérent avec le profil** d'un
  référentiel : massivement lu, peu écrit. Payer à l'écriture pour gagner en lecture, en
  justesse de modélisation et en interrogeabilité polymorphe est le bon échange.
- Le cœur reste ignorant du domaine : il implémente un *mécanisme* d'héritage générique ; il
  n'impose aucune restriction sur *quoi* hérite de *quoi*, laissé libre au métamodèle-client.

## Alternatives écartées

- **Table par classe concrète (feuilles seules), mère dissoute** (« Table-Per-Concrete-
  Class »). Une table par feuille, colonnes de la mère recopiées dans chaque définition, pas
  de table mère. *Rejetée* car elle suppose que seules les feuilles sont instanciables : elle
  n'a **nulle part où stocker un enregistrement créé à un niveau intermédiaire** (un serveur
  sans OS n'a pas de table feuille). Elle ne modélise donc pas correctement le domaine. De
  plus, la requête polymorphe y exige un `UNION` sur toutes les feuilles, lourd quand les
  sous-classes sont nombreuses.
- **Variante « feuilles + vues d'agrégation »**. Matérialiser les feuilles (comme ci-dessus)
  et reconstituer les classes mères par des vues qui `UNION`-nent les filles. *Rejetée* pour
  deux raisons : les vues d'union ne sont pas des cibles d'écriture (ni `INSERT` ni `UPDATE`
  fiable à travers l'union) ; et surtout elle hérite du défaut ci-dessus — elle n'a nulle part
  où instancier un enregistrement de niveau intermédiaire.
- **Table unique par hiérarchie** (« Single-Table Inheritance »). Une seule table pour toute
  la branche, colonne de type, colonnes de toutes les sous-classes réunies (majoritairement
  nulles). Lecture, écriture et requête polymorphe triviales. *Rejetée* car elle produit
  l'**explosion de colonnes** déjà vécue en pratique (classe racine CMDB à plus de 250
  colonnes), ingérable pour une hiérarchie large.

## Conséquences

- **La propagation à l'écriture est de la logique de mutation générée par le cœur, et doit
  être atomique.** Un chemin d'écriture qui oublierait un niveau produirait un enregistrement
  incohérent (présent au niveau spécialisé, absent au niveau mère). Cette orchestration
  multi-tables est un **critère de premier plan pour le choix du moteur GraphQL** (ADR à
  venir) : les moteurs auto-générés gèrent le moins naturellement l'écriture custom
  multi-tables — à peser explicitement contre Hasura / PostGraphile.
- **L'identité partagée entre niveaux est l'invariant critique.** Même identifiant à chaque
  niveau, cohérence garantie (identité générée au niveau racine, clés étrangères entre
  niveaux). Cet invariant DOIT figurer dans `tests/acceptance/` (gelé) : si l'identité se
  désynchronise entre niveaux, le modèle s'effondre silencieusement.
- **La lecture d'un objet complet exige de joindre les niveaux.** C'est le pendant du coût
  d'écriture ; relève de la stratégie anti-N+1 / pushdown à traiter côté moteur et
  résolveurs.
- **La suppression doit elle aussi être propagée** (retirer toutes les lignes d'un
  enregistrement à travers ses niveaux), atomiquement — même exigence que la création.
- **L'évolution du métamodèle** (ajout/retrait d'un niveau ou d'un attribut hérité) reste
  gouvernée par l'expand/contract et les migrations (Principe VI, ADR migrations à venir) :
  cette décision ne modifie pas ce régime, elle en fournit la structure cible.
- **Profondeur et largeur de hiérarchie** : contrairement à la table unique, ce motif ne
  souffre pas de l'explosion de colonnes ; le coût croît avec la *profondeur* (nombre de
  jointures/insertions par enregistrement), pas avec la largeur. Une garde de sanité sur la
  profondeur d'héritage pourra être ajoutée si nécessaire (à décider, non bloquant).