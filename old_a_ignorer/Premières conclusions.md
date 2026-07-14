# Référentiel d'architecture d'entreprise — Relevé de décisions d'architecture

> **Statut du document.** Graine (seed) destinée à être éclatée dans le dépôt en une
> `constitution.md` (principes non négociables) et des ADR (une décision = un fichier).
> Chaque décision ci-dessous est rédigée pour devenir un ADR : *décision / raison /
> alternatives écartées*. Les arbitrages non tranchés sont marqués **OUVERT** et doivent
> voyager avec ce document jusqu'à résolution.

---

## 0. Objet et question de cadrage non résolue

**Objet.** Construire un référentiel d'architecture d'entreprise (applications, capacités
métier, chaînes de valeur, objets métier, projets, et les relations entre eux). Le projet
a démarré comme **banc d'essai pour apprendre le développement assisté par IA**, en
utilisant ce domaine parce qu'il est bien maîtrisé.

**⚠️ Arbitrage fondateur non tranché — à décider avant de fixer le niveau d'exigence.**
Au fil de la conception, l'ambition a glissé vers un produit de qualité production
(volumétrie à ~100 M de lignes envisagée, RLS, multi-cloud, Keycloak). Il faut trancher
explicitement :

- **Banc d'essai** : l'objectif reste d'apprendre *comment l'IA développe*. Niveau
  d'exigence modéré, on privilégie le plus simple qui enseigne.
- **Produit** : on livre un référentiel réel. Niveau d'exigence production, coût et enjeux
  tout autres.

Cette réponse conditionne plusieurs décisions ci-dessous. Tant qu'elle n'est pas posée, on
risque soit de sur-concevoir un apprentissage, soit de sous-concevoir un produit.

**Résultat central de la réflexion.** L'objectif s'est reformulé de « comment l'IA
développe » vers **« où vit le fonctionnel pour survivre à ce que l'IA développe »**. La
stabilité n'est pas un décret, c'est le produit d'une définition précise et isolée.

---

## 1. Principes fondamentaux (seeds de constitution)

1. **La session est jetable ; seuls les artefacts versionnés persistent.** Toute décision
   non écrite dans un fichier commité est perdue. Le contexte est structurellement volatil
   (reboot *et* compaction). Corollaire : le fichier n'est pas l'export d'une décision
   prise ailleurs, c'est le lieu où la décision se prend. Le commit *est* l'acte de décider.

2. **Deux sources de vérité : la spec (intention, prose) et les tests (comportement,
   exécutable).** La plus fiable est la seconde : une spec peut mentir en silence, un test
   échoue bruyamment. Le code est en aval des deux et jetable.

3. **Maximiser le déclaratif, minimiser le bespoke.** Le vrai « moteur stable » (le
   « Glide » du projet) est la stack sur étagère (PostgreSQL, moteur GraphQL), pas le code
   maison. Plus la couche de code volatile est mince, moins la régénération a de surface
   pour nuire.

4. **Sens de dépendance : le code dépend du fonctionnel, jamais l'inverse.** Le fonctionnel
   (métamodèle, schéma, invariants) ne connaît rien du code. Une **couche de contrat**
   explicite, figée, versionnée, propriété humaine, s'intercale entre les deux.

5. **Tout ce qui doit survivre à une régénération se tient hors de son rayon de
   destruction** : propriété humaine, gardé mécaniquement par git, évolué de façon additive
   plutôt que réécrit.

6. **Découpage par volatilité, pas par phase temporelle.** Boucle *cœur* lente et non
   négociable (métamodèle + contrats + schéma de données) ; boucle *périphérie* rapide et
   assumée jetable (UI, glue, connecteurs). Chaque élément est marqué explicitement
   « fondation » ou « spike jetable ». On ne fait pas « tout vite puis on consolide » : la
   capacité à régénérer proprement se construit dès la 1ʳᵉ ligne ou se sacrifie.

7. **Caler la frontière, pas la politique.** Pour tout besoin futur (droits en lecture,
   règles conditionnelles), on pose *l'emplacement* où la règle s'insérera (inerte
   aujourd'hui), sans implémenter la règle elle-même. Évite la sur-conception *et* la
   réécriture ultérieure.

---

## 2. Méthode et outillage

### ADR — Outillage de spécification : Spec Kit sur Claude Code
- **Décision.** Adopter GitHub Spec Kit comme échafaudage par-dessus Claude Code, complété
  de git + un cadre de tests. Pas de framework packagé plus lourd au départ.
- **Raison.** Spec Kit rend l'écriture des artefacts *par défaut* (le fichier est le
  livrable de chaque étape), il est git-natif (revue en PR), neutre vis-à-vis de l'agent
  (une équipe réelle est hétérogène), gratuit, et corrige le symptôme constaté en Claude
  Code nu : les décisions restaient en mémoire de session sans atteindre le disque.
- **Alternatives écartées.**
  - *Claude Code nu* : ne corrige pas le défaut de capture ; discipline entièrement à la
    charge de l'humain.
  - *Superpowers* : plafond « solo/mono-repo », ne répond pas à l'objectif équipe/vraie vie.
  - *BMAD* : organise des **personas d'IA** (analyste, PM, architecte…) qui *simulent* une
    équipe. Redondant ici car les rôles amont sont déjà tenus par des humains (MOA / proxy
    PO). Trop lourd (apprentissage en semaines, coût en tokens élevé), risque d'abandon.
  - *Kiro* : payant et impose de quitter Claude Code.
- **Réserves.** Spec Kit est faible sur l'amendement après coup (re-générer le plan peut
  écraser des éditions manuelles ; versionnement de spec fragile) et ne vérifie pas la
  cohérence *entre* features ni avec le métamodèle. Écosystème « expérimental », taillé
  greenfield, pas cycle de vie complet (rien sur le run/observabilité).

### Garde-fou d'évaluation
En testant l'outillage, séparer « l'outil a mieux capturé » de « l'outil m'a forcé, moi, à
être plus explicite ». Seul le premier est une propriété transférable ; les confondre biaise
la conclusion vers l'optimisme (biais récurrent à surveiller, cf. aussi l'expertise du
concepteur qui comble mentalement les flous, et le « démo-driven » qui fait paraître fini le
facile).

---

## 3. Les trois zones à survie garantie

Tout le reste (`src/`, `ui/`) est régénérable. Ces trois zones sont propriété humaine,
gardées mécaniquement (CODEOWNERS + CI), et **`git diff` doit y être vide après toute
régénération** (vide + suite d'acceptation au vert = régénération propre).

### 3.1 Le contrat de comportement
- Invariants métier et scénarios d'acceptation, **exécutables**, **gelés**, physiquement
  séparés (`tests/acceptance/`), hors du dossier de feature régénéré.
- Distinction cruciale : **tests de contrat** (le *quoi* — propriété humaine, jamais touchés
  par l'agent) vs **tests unitaires d'implémentation** (le *comment* — légitimement
  régénérés avec le code). Le risque majeur : un agent qui régénère code + tests réécrit les
  tests pour coller à ce qu'il vient de produire → suite au vert qui ne prouve rien.
- Les scénarios d'acceptation viennent de l'expert et de la spec, **en amont**,
  indépendamment de l'implémentation (contrepoids au conflit d'intérêt de l'IA qui teste ce
  qu'elle a construit).
- Cliquet : chaque bug de production devient un nouveau test d'acceptation (le filet grandit
  vague après vague). La sûreté de régénération est proportionnelle à la couverture
  d'acceptation, jamais totale.

### 3.2 Les données accumulées
- Schéma + migrations : propriété humaine, **append-only**, **jamais régénérés**. L'agent
  peut *rédiger* une migration ; elle est relue comme critique-production, testée sur des
  données ressemblant à la prod (jamais sur base vide), jamais appliquée automatiquement.
- Motif **expand / contract** (changement parallèle) sur plusieurs livraisons :
  1. *Expand* : ajout additif de la nouvelle structure, l'ancienne coexiste.
  2. *Backfill* : peuplement de la nouvelle structure depuis l'ancienne.
  3. *Transition* : bascule des consommateurs (resolvers, IHM, connecteurs).
  4. *Contract* : suppression de l'ancienne, **dans une livraison ultérieure séparée**.
- Un nouveau champ naît **nullable** ou avec défaut backfillé (jamais `NOT NULL` ajouté à
  une table peuplée sans défaut). GraphQL : on **ajoute** (non-cassant), on **déprécie**
  (`@deprecated`), on ne mute/supprime pas dans la foulée.
- Exigence **transactionnelle** (pas seulement statistique) pour les migrations : chacune
  réussit entièrement ou échoue proprement. Sauvegarde + rollback testés. (Contraste avec
  les tests de comportement, où une fiabilité élevée-mais-partielle est acceptable.)

### 3.3 Le contrat lui-même
- Métamodèle + `schema.graphql` : propriété humaine, versionnés. Le schéma GraphQL **fait
  partie du cœur** (c'est le contrat par lequel tout consommateur touche la donnée), pas un
  simple instrument d'observation du cœur.

---

## 4. Le métamodèle est le produit

- **Décision.** Le métamodèle est un citoyen de première classe, transverse
  (`contracts/metamodele.md`), référencé *par* les specs de features, jamais enfoui dans
  l'une d'elles.
- Il doit être **structuré et lisible par machine** (YAML/JSON), au moins pour ses parties
  porteuses de règles, afin que « le métier configure une règle » devienne « la machine
  l'exécute ».
- **Primauté du métamodèle transverse** sur les `data-model.md` de feature générés par Spec
  Kit (ceux-ci en sont des vues locales). Sinon on recrée deux sources de vérité.
- Le métamodèle **pilote la génération** du `schema.graphql` ET des contraintes de base.
- Distinction à ne pas oublier : ce dépôt contient la **documentation** (specs, ADR,
  métamodèle) — **pas les données** du référentiel (l'inventaire réel des applications,
  etc.), qui vivent dans le magasin applicatif derrière GraphQL.

---

## 5. Arborescence de référence

```
ea-referentiel/
├── CLAUDE.md                    # point d'entrée agent + règles de frontière
├── CODEOWNERS                   # garde git mécanique sur les zones protégées
├── .specify/memory/
│   └── constitution.md          # principes non négociables
│
├── contracts/                   # STABLE, propriété humaine
│   ├── metamodele.md            #   entités, relations, cardinalités, règles, autorisations
│   ├── schema.graphql           #   contrat d'API, versionné
│   ├── glossaire.md
│   └── adr/                     #   une décision = un fichier
│
├── specs/                       # Spec Kit, par feature (quoi/pourquoi)
│   ├── 001-applications/
│   ├── 002-capacites/
│   └── 003-chaines-valeur/
│
├── tests/
│   ├── acceptance/              # CONTRAT DE COMPORTEMENT — gelé, agent interdit
│   │   ├── invariants/
│   │   └── scenarios/
│   └── unit/                    # couplés aux entrailles, régénérés AVEC le code
│
├── migrations/                  # DONNÉES — append-only, propriété humaine, expand/contract
│
├── src/                         # ← RÉGÉNÉRABLE (resolvers, persistence)
└── ui/                          # ← RÉGÉNÉRABLE, ne dépend que du schéma publié
```

Extrait de `CLAUDE.md` (frontières de propriété) :

```
## Frontières de propriété
- contracts/, tests/acceptance/, migrations/ sont PROPRIÉTÉ HUMAINE. Ne jamais les
  modifier. Si un changement y semble nécessaire : STOP, le signaler comme décision
  humaine, ne pas le faire.
- src/ et ui/ sont RÉGÉNÉRABLES à partir de contracts/.
- Toute génération doit satisfaire contracts/schema.graphql à l'identique.
- Le schéma de persistance n'évolue que par une nouvelle migration append-only.
```

> Gardes **molles** (l'agent les respecte la plupart du temps). Garantie **dure** =
> CODEOWNERS + gate CI + `git diff` vide sur les zones protégées. Ne jamais confier la
> frontière à la seule chose qui peut la franchir.

---

## 6. Intégrité des données : où vivent les règles

**Principe de placement.** Chaque règle s'exécute à la couche la plus profonde qui peut
l'exprimer pleinement. La base est la dernière ligne infranchissable (tous les chemins
d'écriture y convergent, y compris ceux qui contournent GraphQL). La frontière GraphQL
ajoute ce que la base ne sait pas dire + les messages lisibles. L'IHM ne fait que du confort.

**Deux natures de règles :**
- **Invariants permanents** (vrais à tout instant) → contraintes de base déclaratives.
  Bonus : plus rapides que la validation applicative (exécution moteur).
- **Validations ponctuelles** (vraies au moment de l'écriture, ex. « date dans le futur »)
  → frontière d'écriture. Piège à éviter : un `CHECK (date > now())` ne se réévalue qu'à
  l'écriture, ne veut pas dire ce qu'on croit.

**Correspondance règle → mécanisme (référence PostgreSQL) :**

| Règle | Frontière GraphQL | Base (ligne infranchissable) |
|---|---|---|
| Champ obligatoire | `String!` | `NOT NULL` |
| Liste de valeurs figée | `enum` | `ENUM` ou `CHECK (x IN …)` |
| Liste éditable à chaud | validation contre table réf. | **table de référence + `FOREIGN KEY`** |
| Liste mère/fille | résolveur | **FK composite** vers table de couples valides |
| Comparaison inter-colonnes (dates) | contrôle croisé | `CHECK (date_fin > date_debut)` |
| Date dans le futur | validation mutation | (déconseillé en base) |
| Format IP / email | scalaire custom | type `inet` / `DOMAIN … CHECK (~ '…')` |

- **Liste éditable à chaud → table + FK, jamais un `enum`.** Ajouter une valeur devient un
  `INSERT` (donnée, à chaud, sans migration) pendant que la FK bloque toute valeur hors
  liste sur tous les chemins. Idem mère/fille (table de couples valides + FK composite).
  C'est le paramétrage fonctionnel voulu, rendu infranchissable et rapide.
- **Génération depuis le métamodèle** pour le reste : une seule source de *définition*, deux
  couches d'*exécution* (défense en profondeur). Le smell serait la duplication de
  définition, que la génération élimine — pas la duplication d'exécution.
- **Triggers = dernier recours** (magie invisible, dur à déboguer), documentés en ADR.

---

## 7. Lecture vs écriture

- Règles d'**écriture** = *correction*. Vivent à la frontière (mutations), volume faible.
- Règles de **lecture** = *performance*. Poussées dans la requête et la base : pushdown des
  filtres/tris (`WHERE`/`ORDER BY`, pas de post-filtrage mémoire), pagination **keyset**
  (curseur, pas `OFFSET`), lutte anti-**N+1** (dataloader/batch — critique car un
  référentiel EA est un graphe), **index** sur colonnes filtrées/triées/jointes.

**Exception importante — l'ingestion en masse (connecteurs/imports)** casse la dichotomie
« écriture non-perf-critique » : elle ne passe pas par le formulaire *et* valide par
milliers. Décision : les connecteurs passent par GraphQL, mais via une **mutation de masse
(bulk)** (un lot, validation ensembliste, rapport de rejets ligne à ligne) — jamais une
mutation par ligne. **[Reporté : sujet imports traité en second temps.]**

---

## 8. Règles spécifiques (le « client script » façon ServiceNow, corrigé)

- La majorité des règles sont **déclaratives** (catalogue de validateurs + liaisons dans le
  métamodèle). Le JavaScript n'est que pour la **queue irréductible** procédurale — pas la
  norme. Mettre en Turing-complet ce qui peut rester déclaratif est une régression de
  gouvernance.
- **Autoritaire côté serveur, projeté (pas délégué) côté navigateur.** Un contrôle
  navigateur est un confort, jamais une autorité : contournable (API, autre canal, console)
  et absent pour un connecteur. Axiome : *le client peut mentir ou être absent ; le serveur
  ne fait jamais confiance à un contrôle amont.*
- Règles déclaratives **sérialisées** : le serveur les exécute (autorité), l'IHM reçoit la
  *même* déclaration qu'un validateur générique rejoue (confort). Une définition, deux
  moteurs, zéro logique dupliquée.
- Queue JS : soit isomorphe (même fonction pure des deux côtés, serveur = autorité en bac à
  sable), soit serveur seul. **Ces scripts sont des règles métier → rangés côté
  `contracts/`, propriété humaine, testés, non régénérables.** Surface de sécurité (bac à
  sable, limites temps/mémoire, pas de secrets/réseau) : sous-système à défendre, à réserver
  au strict nécessaire.

---

## 9. Base de données

### ADR — PostgreSQL
- **Décision.** PostgreSQL.
- **Raison.** Open source ; encaisse les volumes (avec partitionnement/index) ; système de
  contraintes le plus riche (`DOMAIN`, `CHECK`, exclusion, FK) pour porter les invariants ;
  **RLS** disponible ; managé sur tous les clouds (portabilité acquise) ; meilleure cible de
  l'écosystème GraphQL.
- **Alternatives écartées.** Base graphe (Neo4j) : naturelle pour la traversée d'impact,
  mais faible précisément là où est l'exigence (contraintes, intégrité, RLS, gros volumes).
  La traversée se fait en SQL récursif à l'échelle du cœur. *Hedge : extension Apache AGE
  (graphe sur Postgres) si la traversée devient centrale.*

### ADR — Séparer le cœur EA du firehose CMDB
- **Décision.** Ne pas concevoir un magasin uniforme. Cœur EA (milliers à dizaines de
  milliers d'objets curés, gouvernés) et inventaire CMDB (100 M+ de lignes machine, chargées
  en masse, non curées) sont deux sous-systèmes, mêmes si même moteur.
- **Raison.** Appliquer la défense en profondeur (RLS, validation par champ) à 100 M de
  lignes serait inutile et catastrophique en perf. Ne pas laisser la queue CMDB (future et
  incertaine) dicter la conception du cœur. Prévoir la porte, ne pas pessimiser.

---

## 10. Couche GraphQL — **[OUVERT]**

- **Tension à trancher :** code mince (auto-généré) **vs** règles JS + logique de mutation
  custom. L'autorisation d'écriture *par attribut pilotée par le métamodèle* (§11) est le
  premier besoin qui pousse **contre** le pur auto-généré et **vers** une couche de mutation
  maîtrisée.
- **Options :**
  - *Hasura* (Apache 2.0) : très peu de code, RLS/pushdown natifs ; logique très custom
    « à côté » (Actions / connecteurs TypeScript). Permissions par colonne déclaratives
    (couvrent peut-être le cas simple d'aujourd'hui).
  - *PostGraphile* (Node.js) : génère depuis le schéma, s'appuie nativement sur la RLS et
    `pgSettings` ; suppose une aisance Postgres avancé ; moteur Grafast performant.
  - *Serveur fait main* (Apollo/Pothos) : contrôle total, JS/TS natif, mais épaissit la
    couche volatile qu'on veut mince. Déconseillé comme défaut.
- **À trancher** avec 3-4 critères (aisance Postgres de l'équipe, poids de la logique de
  mutation custom, style de génération depuis le métamodèle, câblage JWT→session).

---

## 11. Identité et autorisation

### Séparation fondatrice
- **Authentification** (prouver *qui*) = gros sujet, **reportable** (cible Keycloak).
- **Propagation d'identité + autorisation** (redescendre une identité à la donnée, décider
  ce qu'elle peut faire) = **socle non reportable** (rétro-ajout = réécriture de tous les
  accès). Les deux sont **séparables**.

### ADR — Plomberie d'identité (à poser maintenant, définitive)
- Relais bout en bout : **JWT vérifié** au moteur GraphQL → `SET LOCAL app.user_id = …`
  (+ rôles) dans la **même transaction** que la requête → prédicats en base via
  `current_setting('app.user_id')`.
- `LOCAL` lie la variable à la transaction ⇒ **pas de fuite d'identité** entre requêtes sur
  une connexion mutualisée du pool. **Invariant de sécurité → test d'acceptation gelé**
  (« deux requêtes successives d'utilisateurs différents ne voient jamais le contexte l'une
  de l'autre »).
- La valeur vient du JWT vérifié (non falsifiable) ; l'utilisateur technique garde ses
  droits table, c'est le *contexte* qui porte l'identité.
- **Point de vigilance :** un pooler externe agressif (PgBouncer mode transaction/statement)
  peut casser « même connexion, même transaction » — à vérifier selon le montage.

### ADR — Source d'identité provisoire (démarrage sans Keycloak)
- **Décision.** Au départ, le back émet lui-même un **JWT signé** (clé locale) portant
  `user_id` + `roles`. Keycloak plus tard : on remplace *l'émetteur*, la couche aval est
  intacte (une signature à valider reste une signature à valider).
- **Raison.** Pose le **contrat** (« la couche donnée reçoit un JWT vérifié portant user_id
  et roles ») dès maintenant — c'est lui qui rend Keycloak substituable. On construit la
  moitié aval définitive avec une source amont provisoire, pas du jetable.

### ADR — Indirection de lecture (à poser maintenant, inerte)
- Jamais de tables nues : lecture via une indirection (vue/fonction) dont le prédicat
  d'autorisation est aujourd'hui `TRUE`. Le jour d'un besoin de visibilité différenciée, on
  remplace `TRUE` à un seul endroit, sans toucher schéma/resolvers/clients.

### ADR — RLS non nécessaire au départ
- **Décision.** Pas de RLS initialement. Besoin réel = contrôle d'**écriture** (« qui peut
  modifier quoi »), pas de lecture différenciée (« tout le monde voit tout », volume faible).
- **Raison.** La RLS filtre des **lignes en lecture** ; ce n'est pas le besoin. Le contrôle
  d'écriture se joue à la **frontière (mutations)**, volume faible, identité sous la main.
- *Note :* si un jour la lecture devient différenciée, la RLS reste faisable malgré le pool
  via la variable de session (pas via les rôles Postgres). La plomberie d'identité ci-dessus
  la rend prête.

### ADR — Autorisation d'écriture par (entité, attribut)
- **Décision.** Unité de contrôle = le **couple (entité, attribut)** (un même attribut peut
  être éditable sur une entité et pas sur une autre).
- **Deux natures distinctes** (pas deux valeurs d'un même champ) :
  - `ecriture: systeme` — jamais écrit par un humain quel que soit son rôle (calculé,
    dérivé, alimenté par connecteur). Ne pas modéliser comme « un rôle système » (sinon
    quelqu'un l'obtiendra et corrompra la donnée).
  - `ecriture: <profil>` — éditable, et alors par quels profils.
- **Factorisation par profils nommés** (unité de *déclaration* ≠ unité de *contrôle*) pour
  éviter une matrice géante et divergente :

```yaml
profils_ecriture:
  systeme:    { editable: false }
  metier:     { roles: [metier] }
  technique:  { roles: [technique] }

Application:
  attributs:
    nom:         { type: string, ecriture: metier }
    criticite:   { type: enum(...), ecriture: metier }
    adresse_ip:  { type: inet, ecriture: technique }
    date_modif:  { type: timestamp, ecriture: systeme }
```

- **Forme extensible** : `editable_par` conçu comme une *règle* attachée à l'attribut (pas
  une case cochée), pour absorber demain des conditions contextuelles (« le métier ne
  modifie que les applications de son domaine », « éditable tant que l'objet est brouillon »)
  sans changer la forme.
- **Imposition à la frontière de mutation, sur les seuls attributs réellement modifiés** :
  comparer l'état entrant à l'existant, exiger le droit uniquement sur les attributs changés
  (sinon refus abusif sur un objet renvoyé entier, ou écrasement laxiste). Logique non
  fournie gratuitement par Hasura/PostGraphile.
- **Conséquence assumée de la granularité (entité, attribut) :** aucune garantie
  structurelle de cohérence d'un même concept entre entités (« criticité » sur Application
  vs ObjetMétier sont gouvernées indépendamment). La cohérence transverse devient affaire de
  **revue et convention**, pas de structure.

### ADR — Cible d'authentification : Keycloak (reportée)
- **Décision.** OIDC/OAuth via Keycloak (open source, Docker).
- **Montage.** Authorization Code + PKCE : l'IHM (SPA) redirige vers Keycloak, récupère le
  JWT, l'envoie au back à chaque requête. Le back **vérifie localement** via JWKS (clé
  publique récupérée une fois) — **pas d'appel à Keycloak par requête**.
- **Composants réels à exploiter (≠ juste des conteneurs) :** Keycloak + **sa propre base**
  (persistance obligatoire) + PostgreSQL applicatif + back GraphQL + IHM.
- **Disciplines dès le départ :** config Keycloak **versionnée** (realm import JSON, côté
  `contracts/`) et non cliquée ; secrets hors du compose (`.env` non commité) ; volumes
  nommés explicites ; token mappers décidant *ce que le JWT transporte* (contrat identité).
- **Coût de run assumé :** Keycloak est un point de défaillance unique de l'auth (mises à
  jour parfois cassantes, rotation de clés, sauvegarde, supervision). Le run de Keycloak
  fait partie du run de la solution.

---

## 12. Arbitrages ouverts (doivent voyager avec ce document)

1. **[FONDATEUR] Banc d'essai vs produit de production** (§0) — fixe le niveau d'exigence.
2. **[OUVERT] Hasura vs PostGraphile vs serveur fait main** (§10).
3. **[REPORTÉ] Stratégie d'imports/ingestion de masse** : transaction tout-ou-rien
   **vs** staging façon ServiceNow (lignes valides acceptées, invalides rejetées une à une).
   *Deux philosophies opposées énoncées, non conciliées.*
4. **[FUTUR] Incorporation de la CMDB** (100 M+ lignes) comme sous-système séparé.

---

## 13. Sujets non encore traités (prochaines étapes annoncées)

- **Construction des formulaires et des listes** (IHM, dérivées du métamodèle ?).
- **Build vs buy sur les frameworks** : tout construire, ou s'appuyer sur des frameworks
  existants ? → à décider **contre** les invariants de ce socle, pas contre des souvenirs.
```