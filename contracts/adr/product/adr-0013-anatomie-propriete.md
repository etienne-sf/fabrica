# ADR-0013 — Anatomie de Fabrica et ligne de propriété Fabrica / projet / instance

**Statut :** Proposé — 2026-07-11
**Portée :** décision du **cœur** (Fabrica), générique (Principe IV). Socle dont dépendent les
ADR déploiement à chaud, montées de version, personnalisation. Précise et complète le principe
« Contrat public du cœur » de la constitution. À éclater en
`contracts/adr/0013-anatomie-propriete.md`.

---

## Contexte

Fabrica a été conçue par morceaux (couche serveur, générateur, exécuteur, données système,
catalogues, fonctionnalités) sans jamais réunir la liste de ses parties ni tracer précisément
la frontière entre ce que Fabrica livre et ce qu'un projet ajoute. Cette frontière conditionne
les montées de version, la personnalisation et le déploiement à chaud : on ne peut pas dire
« comment on livre et fait évoluer » sans dire d'abord « qui possède quoi ».

## Décision — trois plans, pas deux

L'architecture se lit sur **trois plans distincts**, à ne pas confondre :

1. **Fabrica — ce qui est livré** (le produit, versionné, non modifiable par le projet).
2. **Le projet — ce qui est défini** (une fois : la définition d'une application). Additif.
3. **L'instance — ce qui est exécuté** (un déploiement concret : dev, recette, prod). Les
   *données* et une partie du *paramétrage* vivent ici, et diffèrent d'une instance à l'autre.

La distinction 2/3 est structurante : le **métamodèle** et les **rôles** sont *définitionnels*
(plan projet, définis une fois) ; les **données** sont *exécutionnelles* (plan instance,
propres à chaque déploiement). Un même projet a plusieurs instances.

## Décision — ce que Fabrica livre

- **Couche serveur** : identité, autorisation, orchestration d'écriture.
- **Générateur** : métamodèle → DDL PostgreSQL + schéma GraphQL (ADR-0011).
- **Exécuteur** : Grafast + PostgreSQL (ADR-0011).
- **Fonctionnalités standard** : exporter, importer, rapports, etc. (livrées par Fabrica ;
  le projet les *consomme*, il n'en crée pas — voir hors périmètre).
- **Données système** : le rôle admin standard, les rôles standards, le seed initial.
- **Format du métamodèle** : le contrat de déclaration (l'API publique d'entrée).
- **Catalogues fermés et extensibles** (propriété de Fabrica) : natures d'ACL, effets d'IHM
  (ADR-0006), conditions de l'axe ligne. Le projet y *pioche*, il ne les étend pas lui-même.

## Décision — ce que le projet définit (additif)

- **Son métamodèle** : entités, attributs, héritage, vues.
- **Ses rôles et ACL** : créés par lui ; il **n'a jamais** à modifier le livré.
- **Ses règles** : effets d'IHM, conditions de ligne — **piochés dans les catalogues** Fabrica.
- **Ses domaines** : regroupements d'entités (organisation du métamodèle ; ADR/spec distinct).

## Décision — points d'extension : la membrane

La ligne de propriété n'est **pas une cloison étanche**, c'est une **membrane** : le projet
la traverse, mais **uniquement par des points d'extension déclarés par Fabrica**.

**Principe directeur (test de tout point d'extension) :** un point d'extension laisse le
projet **composer par-dessus le livré** (hériter, référencer, piocher dans un catalogue),
**jamais le redéfinir ni en connaître l'intérieur**. Test : si étendre exige de comprendre
comment Fabrica est faite à l'intérieur, le point d'extension est mal conçu.
- *Hériter d'un rôle standard* passe le test (on compose sans connaître son contenu).
- *Recopier un rôle standard* échoue (on doit connaître son contenu, et on diverge à la
  première évolution de Fabrica).

C'est la même idée que l'ADR-0007 (« les projets ne dépendent que du contrat, jamais du
moteur »), appliquée à la frontière Fabrica/projet : **on dépend des contrats, jamais des
intérieurs.**

**Héritage des rôles standards (point d'extension exemplaire) :** un rôle de projet **peut
hériter** d'un rôle standard de Fabrica (composition par-dessus, le rôle livré restant
intact). Bénéfice : quand Fabrica fait évoluer son rôle standard, les rôles qui en héritent
en profitent automatiquement — au lieu de diverger comme le feraient des copies.

**Invariant — points d'extension non supprimables :** une fois déclaré, un point d'extension
est un **engagement de compatibilité** (le projet a pu construire dessus). On en *ajoute*, on
n'en *retire* pas — sauf procédure exceptionnelle documentée avec vérification avant
déploiement. Même régime que le contrat public (expand/contract, compatibilité ascendante).

## Hors périmètre (renvois explicites)

- **Fonctionnalités custom du projet** : **non retenues** au démarrage. Permettre au projet de
  *créer* des fonctionnalités impliquerait une API interne d'interaction avec Fabrica — un
  chantier en soi, lié au runtime de règles réservé (ADR-0006). Le projet **consomme** les
  fonctionnalités livrées par Fabrica ; il n'en fabrique pas. (La nature d'ACL
  « fonctionnalité » protège les fonctionnalités *livrées*, ce qui suffit.)
- **Données / instances (plan 3)** : **hors périmètre**. Les données appartiennent à chaque
  instance (dev → prod), pas au projet. Le sujet associé — **clonage d'un environnement sur
  l'autre en conservant le paramétrage** (que propager, que garder propre à chaque instance) —
  est un gros sujet distinct (« données vs configuration »), explicitement réservé.
- **Regroupement des entités en domaines** : organisation du métamodèle, ADR/spec distinct.

## Conséquences

- **La personnalisation est additive par construction** : le projet compose et ajoute, ne
  modifie jamais le livré. Le problème du *merge* (personnalisation client vs mise à jour
  fournisseur) — plaie des progiciels — **disparaît** : une montée de version de Fabrica ne
  touche que les objets Fabrica. (Détaillé dans l'ADR personnalisation.)
- **Les montées de version** ne portent que sur le plan Fabrica ; elles n'écrasent ni les
  définitions du projet, ni les données de l'instance. (Détaillé dans l'ADR données système /
  montées de version.)
- **Le plan instance** ouvre le sujet du clonage/paramétrage, à traiter séparément.
- Cette anatomie impacte l'ADR-0012 : une ACL de projet et une ACL livrée par Fabrica
  cohabitent additivement ; le projet crée les siennes sans toucher celles de Fabrica.
