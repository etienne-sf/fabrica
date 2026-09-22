## ADR-0003 — Contrat d'IHM piloté par le métamodèle (formulaires et listes)

**Statut :** Proposé — 2026-07-11

### Contexte
Les écrans d'édition (formulaires) et de consultation (listes) doivent être dérivés du métamodèle plutôt que codés à la main par entité, tout en permettant une personnalisation utilisateur. Il fallait décider où vit chaque responsabilité pour ne pas laisser du paramétrage fuir dans du code régénérable, ni figer du rendu qui doit rester jetable.

### Décision
Formulaires et listes reposent sur **trois couches distinctes** :
- **Contrat** (métamodèle, versionné, propriété humaine) : structure du formulaire, colonnes par défaut de la liste, bornes de personnalisation. Ne re-déclare aucun attribut — il *sélectionne et ordonne* des attributs déjà déclarés.
- **Préférence** (donnée utilisateur, en base, par utilisateur) : colonnes choisies et ordre.
- **Rendu** (périphérie, régénérable) : lit contrat + préférence et affiche.

**Déclarations.**
- `label` et `listable` sont portés par **l'attribut** (source unique, consommée par formulaire et liste). L'ensemble des colonnes offertes se **dérive** des attributs `listable: true` — pas de liste `colonnes_disponibles` re-déclarée.
- La vue `liste` déclare `colonnes_defaut` et une `personnalisation` **bornée** : `colonnes_verrouillees` (jamais retirables), `max_colonnes` (borne de perf), `ajout_retrait`, `reordonnancement`.
- Contrainte de cohérence imposée par la validation du métamodèle : toute colonne par défaut ou verrouillée doit référencer un attribut `listable: true`.

**Sémantique de composition : option A stricte.**
- *Préférence absolue* : la préférence est la liste complète et ordonnée voulue (pas un différentiel).
- *Pas de propagation des nouveaux défauts* : un utilisateur ayant une préférence enregistrée reste sur sa vue ; une colonne ajoutée aux `colonnes_defaut` en v2 n'apparaît pas chez lui.
- *Filtrage systématique contre le métamodèle courant à l'affichage* (non optionnel) : à la lecture, la préférence est **intersectée** avec les attributs actuellement `listable`. Une colonne référençant un attribut disparu ou passé `listable: false` est ignorée silencieusement ; ne surpasse jamais `colonnes_verrouillees` ni `max_colonnes`. **Le métamodèle est autoritaire, la préférence subordonnée.**

**Autorisation, deux axes distincts.**
- *Édition de la donnée* : réutilise **sans exception** l'autorisation d'écriture par (entité, attribut) — cf. constitution, section « Discipline de développement » (règle `ecriture:`), et le futur ADR « Autorisation d'écriture par attribut » [à créer]. Formulaire et cellule de liste sont deux vues du même contrat de droits. Le rendu interroge l'éditabilité **par cellule** (ligne × attribut × utilisateur), **sans** mettre en cache « colonne éditable » — pour absorber sans refonte les règles conditionnelles futures.
- *Édition de la préférence* : un utilisateur n'écrit que **sa propre** préférence (`user_id` = utilisateur authentifié). Permission distincte, sur un objet distinct.

**Frontière d'écriture.** L'édition en liste passe par la **même** mutation, la même
validation et la même autorisation que le formulaire — cf. futurs ADR « Placement des règles / validation » et « Autorisation d'écriture par attribut » [à créer]. La liste n'est pas un canal d'écriture privilégié.

### Alternatives écartées
- *Option B (préférence différentielle)* : propagerait les nouveaux défauts, mais complexité forte sur l'ordonnancement exprimé en diff, pour un gain jugé non impératif.
- *Option A brute (sans filtrage)* : casse à la première évolution du métamodèle (références mortes). Rejetée.
- *Liste `colonnes_disponibles` re-déclarée par vue* : deux listes à maintenir qui divergent. Remplacée par la dérivation depuis `listable`.

### Conséquences
- Le métamodèle doit porter, par attribut, au moins : type, contraintes, `ecriture:`, `label`, `listable` ; et par vue : structure de formulaire, `colonnes_defaut`, bornes de personnalisation. Ceci alourdit le contrat central (cf. ADR « Contrat de déclaration du métamodèle » [à écrire, prochaine spec]).
- **Point de perf reporté** : une colonne personnalisée pointant vers un attribut d'entité liée (ex. « nom du propriétaire ») retombe sur le pushdown / anti-N+1 ; la personnalisation est un générateur de requêtes dynamiques, à traiter comme tel à l'implémentation.
