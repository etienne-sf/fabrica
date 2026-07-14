# ADR-0011 — Moteur GraphQL : schéma généré par Fabrica, exécution par Grafast

**Statut :** Proposé — 2026-07-11
**Portée :** décision du **cœur** (Fabrica), générique (Principe IV). Débloque plusieurs ADR
qui la présupposaient (0005 orchestration d'écriture, 0007 isolation, 0010 propagation
d'identité). À éclater en `contracts/adr/0011-moteur-graphql.md`.

---

## Contexte

Question longtemps différée : quel moteur GraphQL au-dessus de PostgreSQL ? Elle a été
reformulée en distinguant les **briques** réellement en jeu, le mot « moteur » étant ambigu :

- **B1** application-cliente (SPA du projet, consommateurs externes) ;
- **B2** schéma GraphQL public (la *forme* de l'API : types, champs, pagination, erreurs) —
  c'est ce que B1 consomme ;
- **B3** couche Fabrica (identité, autorisation, orchestration) ;
- **B4** générateur d'API (fabrique un schéma GraphQL à partir du schéma de base, en imposant
  ses conventions) ;
- **B5** exécuteur (compile GraphQL → SQL, optimise : pushdown, anti-N+1, pagination) ;
- **B6** PostgreSQL.

Un « moteur GraphQL » complet (PostGraphile, Hasura) fournit B4 **et** B5 couplés. Deux
décisions amont ont resserré le choix : l'option **C** a été retenue (Fabrica produit B2
elle-même, depuis le métamodèle — cf. ci-dessous), ce qui pose la question : puisque Fabrica
possède B2, un moteur complet est-il encore utile ? « Exécuter du GraphQL contre Postgres » se
décompose en trois problèmes : (1) parser/valider — résolu par les bibliothèques GraphQL
standard ; (2) traduire en SQL ; (3) **optimiser** (anti-N+1, pushdown) — le seul réellement
difficile, cœur d'un référentiel qui est un graphe d'entités liées.

## Décision

**Fabrica génère elle-même le schéma GraphQL public (B2) à partir du métamodèle. L'exécution
est confiée à Grafast (B5), avec `@dataplan/pg` comme pont vers PostgreSQL. On n'utilise
aucun moteur GraphQL complet : ni le générateur d'API de PostGraphile (B4), ni Hasura.**

Architecture résultante :
- **B4 supprimé.** Le générateur d'API d'un moteur tiers n'est pas utilisé.
- **B3 (Fabrica) produit B2** depuis le métamodèle : c'est un générateur de schéma GraphQL à
  part entière, appartenant au cœur. Il permet aussi d'exposer des **directives et
  conventions propres à Fabrica**, dérivées du métamodèle, qu'aucun générateur de moteur tiers
  n'aurait permises.
- **B5 = Grafast + `@dataplan/pg`.** Grafast est utilisé *seul*, en remplacement de la fonction
  d'exécution de GraphQL.js, pour un schéma que Fabrica définit — c'est un mode d'usage
  explicitement supporté par Graphile (« construire son propre schéma et obtenir la meilleure
  performance sans effort supplémentaire »). Les résolveurs sont écrits comme *plan resolvers*
  Grafast. `@dataplan/pg` porte l'interaction SQL efficace.
- **B6** PostgreSQL, inchangé (ADR-0004).

Le couplage redouté entre B4 et B5 (jeter le générateur ferait perdre l'exécuteur) **n'existe
pas** : Graphile a séparé Grafast de la génération de schéma de PostGraphile. On garde donc
l'exécuteur sans garder le générateur.

## Raison

- **Isolation (ADR-0007) acquise par construction.** Puisque Fabrica produit B2, aucune
  convention de moteur ne transparaît dans les messages GraphQL (requêtes/réponses) consommés
  par les projets. La gêne « le moteur façonne mon contrat public » disparaît : le contrat est
  100 % à Fabrica.
- **Métamodèle source unique.** Le métamodèle pilote déjà le DDL (B6) ; il pilote désormais
  aussi B2. Une seule source pour tout ce qui est public (base + API), cohérent avec l'esprit
  métamodèle-piloté du projet.
- **On ne réécrit pas le problème dur.** L'optimisation (planification, anti-N+1, pushdown)
  est le seul problème réellement difficile ; Grafast le résout et représente des années de
  maturation (V5 est une réécriture complète autour de ce moteur de planification). Le porter
  soi-même serait un investissement disproportionné.
- **Compatibilité native avec l'héritage (ADR-0005).** Grafast/PostGraphile V5 gère la
  polymorphie relationnelle — une table centrale (interface + attributs partagés) jointe
  une-à-une à une table par type pour les attributs additionnels : c'est exactement la
  matérialisation table-par-classe de l'ADR-0005. L'exécuteur sait déjà exécuter ce modèle.
- **Standards maîtrisés, non subis.** Pagination, nommage, identifiants : ce ne sont plus des
  conventions imposées par un générateur tiers, mais des choix que Fabrica pose en générant
  B2 (au prix d'en porter la responsabilité).

## Alternatives écartées

- **Option A — le générateur du moteur (B4) produit B2 exposé aux projets** (mode PostGraphile
  ou Hasura par défaut). Simple et gratuit, mais couple les projets aux conventions du moteur ;
  un changement de moteur casse les applications-clientes. Contredit l'ADR-0007. Rejetée.
- **Option B — garder B4, le masquer derrière une couche de traduction dans B3.** Évite
  d'écrire un générateur, mais impose une couche de traduction à maintenir et laisse une
  dépendance résiduelle aux conventions de B4. Rejetée au profit de C : le générateur propre
  est plus cohérent (métamodèle source unique) et permet les directives propres à Fabrica.
- **Hasura.** Produit qui génère *et* exécute de façon couplée ; n'expose pas un exécuteur
  qu'on pilote avec son propre schéma. Incompatible avec l'option C telle que réalisée.
  Licence/trajectoire SaaS par ailleurs en tension avec l'exigence open source ferme. Rejetée.
- **PostGraphile complet (B4+B5).** Fournirait le générateur, mais c'est précisément ce qu'on
  veut posséder. Rejeté au profit de Grafast seul.
- **Tout porter soi-même (parser + traduction SQL + optimisation).** Contrôle total, mais
  réécrit l'optimisation anti-N+1 — le morceau dur et mûr de Grafast. Investissement
  disproportionné. Rejetée.

## Coût assumé (à ne pas masquer)

- **Fabrica porte un générateur de schéma GraphQL (métamodèle → B2) ET des plan resolvers
  Grafast.** C'est un composant *substantiel* du cœur — celui que PostGraphile fournissait en
  B4, réécrit pour le posséder. La maîtrise du GraphQL réclamée se paie en code que Fabrica
  possède et maintient. Cohérent avec l'architecture (« Fabrica épais, projets minces »), mais
  d'une ampleur notable.
- **Couplage interne à Grafast.** Les plan resolvers sont spécifiques à l'écosystème Grafast ;
  ils ne sont pas portables vers un autre exécuteur (Hasura a un modèle tout autre). Ce
  couplage est **interne à Fabrica (B3)** et **jamais exposé aux projets** (grâce à C). Un
  changement d'exécuteur serait donc un chantier interne réel — porté une fois par le
  fournisseur du socle — sans aucun impact sur les projets-clients.

## Conséquences

- **L'orchestration d'écriture multi-tables atomique (ADR-0005)** vit dans B3 : les mutations
  sont des plan resolvers / opérations `@dataplan/pg` exécutées dans une transaction que
  Fabrica contrôle. Cohérent avec « le custom d'écriture est porté par la couche ».
- **La propagation d'identité (ADR-0010)** repose sur l'injection par transaction
  (`SET LOCAL`) au moment de l'exécution : à réaliser via le mécanisme de réglages de session
  de l'écosystème Grafast/`@dataplan/pg` (équivalent `pgSettings`). Point à confirmer à
  l'implémentation ; lève le point de vigilance « compatibilité moteur » de l'ADR-0010.
- **Le bus-factor** (fil de discussion) est ramené à un risque mineur : Grafast n'est plus
  qu'un exécuteur interne, remplaçable sans impact projet. Le projet Graphile reste porté par
  une petite équipe (financement par sponsors, dont de grands acteurs), mais il est
  **entièrement open source, auto-hébergeable (process Node.js / middleware) et forkable** : le
  risque « perdre le support » subsiste, le risque « perdre l'outil » est nul.
- **Les standards (pagination, identifiants, format d'erreurs)** deviennent une responsabilité
  de Fabrica, à fixer dans la génération de B2 — et à verser au contrat public versionné
  (constitution, compatibilité ascendante).

## Points à confirmer à l'implémentation

- Injection `SET LOCAL` par requête via l'écosystème Grafast (mécanisme de session variables).
- Couverture, par le générateur de B2, de tous les cas que le métamodèle peut exprimer
  (types, relations, héritage polymorphe, filtres, pagination) — c'est la charge principale.
- Ergonomie des plan resolvers pour l'orchestration d'écriture multi-niveaux (ADR-0005).