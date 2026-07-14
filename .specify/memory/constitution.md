<!--
SYNC IMPACT REPORT
==================
Version change: (template, non versionné) → 1.0.0
Type de changement: ratification initiale (corrigée avant mise en dépôt).

Note sur la version. Une première rédaction datée du même jour, générée par le skill
`/speckit.constitution`, avait relu d'anciens fichiers de cadrage traînant dans le dépôt
et en avait tiré un OBJET DE PROJET PÉRIMÉ (« référentiel d'architecture d'entreprise »),
en contradiction directe avec le Principe IV. Ce brouillon n'ayant jamais été commité ni
rendu opérant, sa correction relève de la ratification initiale, non d'un amendement d'une
version publiée. La version reste donc 1.0.0. Si le porteur considère que le brouillon
faisait foi, cette correction devient un MAJOR (redéfinition de l'objet) → 2.0.0.

Principes définis (7) — inchangés par rapport au brouillon:
  - I.   La session est jetable ; seuls les artefacts versionnés persistent
  - II.  Deux sources de vérité : la spec et les tests
  - III. Maximiser le déclaratif, minimiser le bespoke
  - IV.  Le code dépend du fonctionnel, jamais l'inverse
  - V.   Hors du rayon de destruction
  - VI.  Découpage par volatilité, pas par phase
  - VII. Caler la frontière, pas la politique

Corrections apportées au brouillon:
  - OBJET DU PROJET redéfini : Fabrica est un moteur générique piloté par métamodèle,
    pas un référentiel d'EA. L'EA est le premier métamodèle-client. (Corrige la
    contradiction avec le Principe IV.)
  - Mentions « banc d'essai » remplacées par « banc de validation » (rôle résiduel de l'EA).
  - Section « Portée » réécrite : acte le cadrage produit-socle réutilisable, la fenêtre de
    régénération / bascule, et la décision (+ risque assumé) du banc de validation EA.
  - Section AJOUTÉE : « Contrat public du cœur et compatibilité ascendante » (dimension
    plateforme versionnée, absente du brouillon).
  - Contrôle de conformité : ajout d'une vérification explicite du Principe IV (le cœur ne
    nomme aucun concept de domaine).

Clarifications ultérieures (avant commit, toujours en ratification 1.0.0):
  - Principe IV : précisé que la prohibition de nommer un concept de domaine vise le CODE du
    cœur, non le métamodèle-donnée (un métamodèle EA hébergé pour le banc nomme légitimement
    ses concepts).
  - Portée › banc de validation : distinction explicite des deux usages de l'EA — (a) le
    métamodèle EA comme donnée d'entrée du banc dans le dépôt du cœur, (b) le produit EA de
    production comme projet-client dans un dépôt distinct.

Sections retirées: aucune.

Modèles à vérifier:
  - .specify/templates/plan-template.md ............ ✅ aligné (« Constitution Check » lit
    dynamiquement ce fichier — aucun nom de principe codé en dur)
  - .specify/templates/spec-template.md ............ ✅ aligné (aucune référence à un
    principe précis)
  - .specify/templates/tasks-template.md ........... ✅ aligné (catégories génériques)

TODO différés:
  - ADR à écrire : « Banc de validation fondé sur le métamodèle EA réel » (décision +
    risque de non-détection de contamination, cf. Portée › Banc de validation).
  - ADR à écrire : « Stratégie d'extraction du cœur en paquet importable » (cf. Contrat
    public du cœur).
-->

# fabrica Constitution

> **fabrica** — moteur générique piloté par métamodèle. Il transforme la déclaration d'un
> métamodèle (entités, attributs, contraintes, autorisations, vues) en application
> fonctionnelle (schéma de base, API GraphQL, formulaires, listes). Fabrica ne connaît
> aucun domaine applicatif : les métamodèles sont ses **données d'entrée**. Le référentiel
> d'architecture d'entreprise est le **premier métamodèle-client** de Fabrica et sert de
> banc de validation et de vitrine ; il vivra dans un dépôt distinct. Cette constitution
> fixe les règles non négociables qui rendent le cœur stable **malgré** ce que l'IA régénère.

## Core Principles

### I. La session est jetable ; seuls les artefacts versionnés persistent

Toute décision non écrite dans un fichier commité est réputée perdue : le contexte est
structurellement volatil (reboot de session *et* compaction). Le fichier n'est PAS l'export
d'une décision prise ailleurs — il EST le lieu où la décision se prend, et le commit EST
l'acte de décider.

**Règles.** Toute décision d'architecture ou de conception DOIT exister comme fichier
versionné (spec, ADR, métamodèle, contrat) avant d'être considérée comme actée. Une
décision qui n'a atteint que la conversation n'engage rien. Rationale : ce qui vit
uniquement en mémoire de session n'atteint jamais le disque et disparaît sans bruit.

### II. Deux sources de vérité : la spec et les tests

L'intention vit dans la **spec** (prose) ; le comportement vit dans les **tests**
(exécutable). Des deux, le test est le plus fiable : une spec peut mentir en silence, un
test échoue bruyamment. Le code est en aval des deux et jetable.

**Règles.** Le comportement attendu DOIT être capturé par des tests d'acceptation
exécutables, distincts des tests unitaires d'implémentation. Les tests de **contrat** (le
*quoi*) sont propriété humaine et NE DOIVENT JAMAIS être réécrits par un agent ; les tests
unitaires (le *comment*) sont légitimement régénérés avec le code. Les scénarios
d'acceptation viennent de l'expert et de la spec, **en amont** de l'implémentation.
Rationale : un agent qui régénère code + tests réécrit les tests pour coller à ce qu'il
vient de produire → une suite au vert qui ne prouve rien.

### III. Maximiser le déclaratif, minimiser le bespoke

Le moteur stable du projet est la stack sur étagère (PostgreSQL, moteur GraphQL), pas le
code maison. Plus la couche de code volatile est mince, moins la régénération a de surface
pour nuire.

**Règles.** Une règle exprimable de façon déclarative (contrainte de base, permission
déclarative, validateur catalogué lié au métamodèle) NE DOIT PAS être codée en impératif.
Le JavaScript/procédural est réservé à la queue irréductible, rangée côté `contracts/`,
testée et non régénérable. Mettre en Turing-complet ce qui peut rester déclaratif est une
régression de gouvernance à refuser.

### IV. Le code dépend du fonctionnel, jamais l'inverse

Le fonctionnel (métamodèle, schéma, invariants) ne connaît rien du code applicatif. Une
**couche de contrat** explicite, figée, versionnée, propriété humaine, s'intercale entre les
deux.

**Règles.** La dépendance va du projet/du code VERS le cœur, jamais l'inverse ; le cœur ne
contient aucune connaissance d'un domaine applicatif particulier — les métamodèles sont des
**données d'entrée**, jamais du code du cœur. Aucun fichier du cœur ne DOIT nommer un concept
de domaine (p. ex. « application », « capacité », « chaîne de valeur ») : un tel nom est une
fuite à corriger. Cette prohibition vise le **code** du cœur (`src/`, `ui/`, générateurs et
tout artefact régénérable), NON le métamodèle-donnée : un métamodèle EA hébergé dans le dépôt
pour le banc de validation nomme légitimement ses concepts, car il est *entrée*, pas cœur. Le
métamodèle transverse (`contracts/metamodele.md`) prime sur les vues
locales de feature ; il pilote la génération du `schema.graphql` et des contraintes de base.
Toute génération DOIT satisfaire `contracts/schema.graphql` à l'identique. Rationale : sans
ce sens de dépendance imposé, on recrée deux sources de vérité et le fonctionnel devient
l'otage du code ; et un cœur qui « connaît » l'EA n'est pas réutilisable.

### V. Hors du rayon de destruction

Tout ce qui doit survivre à une régénération se tient **hors de son rayon de destruction** :
propriété humaine, gardé mécaniquement par git, évolué de façon additive plutôt que réécrit.

**Règles.** Les zones `contracts/`, `tests/acceptance/` et `migrations/` sont propriété
humaine et ne sont JAMAIS modifiées par un agent. La garantie est **dure** (CODEOWNERS +
gate CI + `git diff` vide sur ces zones après toute régénération), jamais confiée à la seule
chose qui peut franchir la frontière. `src/` et `ui/` sont régénérables à partir de
`contracts/`. Rationale : on ne confie pas la frontière à l'agent qui a intérêt à la
franchir ; la sûreté vient de la mécanique, pas de la bonne volonté.

### VI. Découpage par volatilité, pas par phase temporelle

Le projet distingue une boucle **cœur** lente et non négociable (métamodèle + contrats +
schéma de données) d'une boucle **périphérie** rapide et assumée jetable (UI, glue,
connecteurs).

**Règles.** Chaque élément DOIT être marqué explicitement « fondation » ou « spike
jetable ». On NE fait PAS « tout vite puis on consolide » : la capacité à régénérer
proprement se construit dès la première ligne ou se sacrifie. Le schéma de persistance
n'évolue que par une migration **append-only** (motif expand → backfill → transition →
contract, le contract dans une livraison ultérieure séparée) ; une migration est relue comme
critique-production, testée sur données ressemblant à la prod, jamais appliquée
automatiquement. Rationale : la volatilité, pas le calendrier, décide du niveau de soin.

### VII. Caler la frontière, pas la politique

Pour tout besoin futur (droits en lecture différenciée, règles conditionnelles, RLS,
imports de masse), on pose *l'emplacement* où la règle s'insérera — inerte aujourd'hui —
sans implémenter la règle elle-même.

**Règles.** Les points d'extension non reportables DOIVENT être posés dès maintenant sous
forme inerte : indirection de lecture au prédicat `TRUE`, plomberie d'identité
(JWT vérifié → `SET LOCAL app.user_id` dans la même transaction), autorisation d'écriture
modélisée comme *règle* attachée au couple (entité, attribut) plutôt que comme case cochée.
On NE code PAS la politique tant que le besoin n'est pas réel. Rationale : poser la frontière
évite à la fois la sur-conception (implémenter une règle absente) et la réécriture ultérieure
(rétro-ajout = réécriture de tous les accès).

## Zones à survie garantie et frontières de propriété

Trois zones sont propriété humaine, gardées mécaniquement, et leur `git diff` DOIT être vide
après toute régénération (vide + suite d'acceptation au vert = régénération propre) :

1. **Le contrat de comportement** — `tests/acceptance/` : invariants et scénarios
   exécutables, gelés, hors du dossier de feature régénéré. Chaque bug de production devient
   un nouveau test d'acceptation (cliquet : le filet grandit vague après vague).
2. **Les données accumulées** — `migrations/` : schéma et migrations append-only, jamais
   régénérés ; exigence **transactionnelle** (chaque migration réussit entièrement ou échoue
   proprement, sauvegarde + rollback testés).
3. **Le contrat lui-même** — `contracts/` : métamodèle + `schema.graphql` versionnés. Le
   schéma GraphQL fait partie du cœur (c'est le contrat par lequel tout consommateur touche
   la donnée), pas un simple instrument d'observation.

`CODEOWNERS` + gate CI constituent la garantie dure. `CLAUDE.md` porte les gardes molles
(frontières de propriété rappelées à l'agent) ; en cas de changement nécessaire dans une
zone protégée : STOP, signaler comme décision humaine, ne pas le faire.

## Discipline de développement et intégrité des données

- **Outillage.** GitHub Spec Kit par-dessus Claude Code + git + un cadre de tests : le
  fichier est le livrable par défaut de chaque étape, la revue passe en PR, l'outillage
  reste neutre vis-à-vis de l'agent. Réserve connue : Spec Kit est faible sur l'amendement
  après coup (re-générer peut écraser des éditions manuelles) et ne vérifie pas la cohérence
  inter-features ni avec le métamodèle — vigilance humaine requise.
- **Placement des règles.** Chaque règle s'exécute à la couche la plus profonde qui peut
  l'exprimer pleinement. La base est la dernière ligne infranchissable (tous les chemins
  d'écriture y convergent) ; la frontière GraphQL ajoute ce que la base ne sait pas dire +
  les messages lisibles ; l'IHM n'est que du confort. Une liste éditable à chaud → table de
  référence + FK, jamais un `enum`.
- **Serveur autoritaire, client projeté (jamais délégué).** Un contrôle navigateur est un
  confort contournable et absent pour un connecteur ; le serveur ne fait jamais confiance à
  un contrôle amont. Une définition de règle, deux moteurs d'exécution (défense en
  profondeur), zéro logique dupliquée.
- **Autorisation d'écriture par (entité, attribut)**, imposée à la frontière de mutation sur
  les seuls attributs réellement modifiés (comparaison entrant/existant). `ecriture: systeme`
  et `ecriture: <profil>` sont deux natures distinctes, pas deux valeurs d'un même champ.

## Contrat public du cœur et compatibilité ascendante

Fabrica étant un produit-socle réutilisable (cf. Portée), le **format de déclaration du
métamodèle** — et ce que la génération en produit — constitue l'**API publique** de la
plateforme, consommée par des projets-clients successifs. À ce titre :

- **Le contrat du cœur est versionné (sémantique) vis-à-vis de ses clients** : additif =
  mineur, cassant = majeur. Le motif expand → contract du Principe VI s'applique au contrat
  du cœur lui-même, pas seulement aux données : une nouvelle version du cœur NE DOIT PAS
  casser un métamodèle-client qui tournait sur la précédente.
- **Le cœur est conçu dès maintenant comme un paquet importable** : aucune dépendance
  sortante du cœur vers un projet-client ; la dépendance est à sens unique (client → cœur).
  Un projet-client *importe* une version épinglée du cœur, il ne le *contient* pas.
- **Chaque projet-client épingle une version du cœur** et adopte les montées de version à
  son rythme ; les bugs et manques du cœur découverts en développant un client se corrigent
  dans le cœur, publiés en nouvelle version, puis remontés côté client — jamais par copie
  divergente du code du cœur.

Ces disciplines ne s'activent pleinement qu'à partir de la bascule décrite en Portée ; avant
elle, le cœur est un spike régénérable.

## Governance

Cette constitution prime sur toute autre pratique. En cas de conflit entre un artefact
(spec, plan, tâche, code) et un principe ci-dessus, le principe l'emporte et l'artefact est
corrigé.

**Portée — nature du projet.** Fabrica est un **produit-socle réutilisable** : un moteur
générique piloté par métamodèle, destiné à servir plusieurs projets-clients successifs. Le
référentiel d'architecture d'entreprise en est le **premier client** et sert de banc de
validation et de vitrine ; il ne définit pas la nature de Fabrica. Ce cadrage fixe le niveau
d'exigence : celui d'un composant dont d'autres projets dépendront, et non celui d'un
livrable unique ou d'un prototype jetable. Un défaut du cœur se propage à tous ses clients ;
l'exigence de stabilité, de compatibilité ascendante et de documentation du contrat
s'apprécie à cette aune.

**Portée — fenêtre de régénération et bascule.** Tant qu'aucun projet-client réel ne dépend
du cœur, celui-ci reste un *spike* que l'on peut régénérer intégralement. À partir de la
première version publiable — consommée par un projet distinct — le cœur cesse d'être
régénérable et n'évolue plus que par amendement additif et versionné (cf. section « Contrat
public du cœur et compatibilité ascendante »). Cette bascule est un **événement
observable** : la naissance d'un second dépôt, celui du référentiel EA de production, qui
*importe* le cœur versionné au lieu de le contenir.

**Portée — banc de validation, décision et risque assumé.** Deux usages distincts de l'EA
coexistent et NE DOIVENT PAS être confondus : (a) le **métamodèle EA**, *donnée d'entrée* du
banc de validation, hébergé dans le dépôt du cœur ; (b) le **produit EA de production**, futur
projet-client qui vivra dans un dépôt distinct et *importera* le cœur versionné (cf. la bascule
ci-dessus). Le banc de test intégré au dépôt du cœur est fondé sur le **métamodèle EA réel**
(et non sur un métamodèle-jouet dédié).
Conséquence explicitement acceptée : la généricité du cœur n'est pas prouvée par un second
métamodèle dissemblable ; une hypothèse spécifique à l'EA pourrait remonter dans le cœur sans
être détectée par le seul banc EA. Ce risque est jugé acceptable au regard du coût d'un
second fonctionnel à maintenir. La garde qui subsiste est le Principe IV (le cœur ne nomme
aucun concept de domaine) ; sa vérification reste **humaine**, à défaut de canari
automatique. *(Décision tracée en ADR.)*

**Amendements.** Tout amendement se fait par édition versionnée de ce fichier, en PR, avec
mise à jour du Sync Impact Report en tête de fichier et propagation aux modèles dépendants
(`.specify/templates/*.md`). Une décision d'architecture ponctuelle relève d'un ADR
(`contracts/adr/`, une décision = un fichier) ; seuls les principes non négociables vivent
ici.

**Versionnement (sémantique).**
- **MAJOR** : retrait ou redéfinition incompatible d'un principe ou d'une règle de
  gouvernance.
- **MINOR** : ajout d'un principe/section, ou extension matérielle d'une directive.
- **PATCH** : clarification, reformulation, correction non sémantique.

**Contrôle de conformité.** Chaque plan de feature passe une porte « Constitution Check »
(cf. `plan-template.md`) avant la phase de conception, re-vérifiée après. Cette porte vérifie
notamment le Principe IV : aucun concept de domaine applicatif ne remonte dans le cœur. Toute
PR touchant `src/` ou `ui/` vérifie que le `git diff` des zones protégées est vide et que la
suite d'acceptation est au vert. Toute dérogation à un principe DOIT être justifiée dans la
section « Complexity Tracking » du plan, avec l'alternative plus simple explicitement écartée.

**Version**: 1.0.0 | **Ratified**: 2026-07-11 | **Last Amended**: 2026-07-11