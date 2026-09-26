# TOOL-0003 — Mise à disposition du code projet, arborescence du dépôt, git et exécution locale

**Statut :** Proposé — 2026-07-11
**Famille :** **Outillage** (`tool/`). **Versant produit :** mécanisme de script (ADR product-0031) —
cet ADR dit *comment le code projet arrive jusqu'au sandbox*. S'appuie sur tool-0001 (git source de
vérité, format) et tool-0002 (revue de montée de version). Générique (Principe IV).

---

## Contexte

L'ADR product-0031 pose comment Fabrica **exécute** un script (sandbox, contexte, contrat). Il reste à
définir comment le code du projet est **créé, rangé, versionné, compilé et testé**, et plus
largement l'**arborescence du dépôt projet** qui héberge métamodèle, scripts et seed.

## Décision — Fabrica crée le fichier de script

- C'est **Fabrica qui crée le fichier** d'un script (squelette : bon emplacement, bon nom, signature
  conforme au contrat du point d'entrée). La **liaison** métamodèle↔script est ainsi correcte **par
  construction** — un fichier créé à la main n'offrirait aucune garantie.
- **Édition via Fabrica**, avec un éditeur de code existant intégré (**Monaco**, l'éditeur de VS Code —
  *buy*, pas *build*). **Le fichier reste la vérité** : Fabrica l'écrit dans l'arbre de travail git ;
  l'éditer aussi dans un IDE reste possible. La sécurité de la liaison vient de la création par
  Fabrica et de la vérification de complétude, pas d'une interdiction d'éditer ailleurs.
- **Code séparé, pas inline (option B)** : le script est un fichier `.ts`, jamais du code dans le JSON
  du métamodèle (qui perdrait sa lisibilité en diff et sa merge-abilité — tool-0001). Il est rangé
  **au plus près de son usage** : dans le répertoire de l'entité.
- **Typage du contexte sans import** : Fabrica **génère des déclarations de types** par entité (types
  de l'objet courant et de ses attributs). Ce sont des **types seulement, effacés à la compilation** —
  pas des imports d'exécution — donc la règle « script autonome, sans import » du sandbox (product-0031)
  tient, tout en donnant autocomplétion et contrôle de conformité. Dérivés du métamodèle, ces types
  **ne sont pas versionnés** (régénérés dans `.fabrica/`).

## Décision — la partie Fabrica n'entre pas dans le projet

- Le dépôt projet ne contient **aucun fichier de Fabrica**. Il **épingle la version** de Fabrica utilisée
  (`fabrica.json`), comme une dépendance. Méta-métamodèle, entités système, traductions `system:`,
  catalogues, gabarits vivent dans le paquet/l'image Fabrica, **en lecture seule** : le projet ne peut
  pas les modifier, puisqu'il ne les possède pas. Au packaging, l'image de base est la version épinglée.
- **Exception 1 — gabarits appliqués** : ils entrent dans le projet comme **copies possédées par le
  projet** (constitution, « Gabarits fournis »).
- **Exception 2 — surcharges d'entités Fabrica** : le projet peut **paramétrer une entité Fabrica sans
  jamais contenir sa définition**, dans `fabrica-entities/<code>/` (fichiers projet uniquement, jamais
  d'`entity.json`). **MVP** : surcharge des **niveaux de traçabilité** et des **ACL**. **Pas** des vues.
- **Réservé (hors MVP)** : **ajouter des attributs** aux entités système (ex. propriétés des
  utilisateurs propres à l'organisation) — ce qui tirera le paramétrage des vues, formulaires et effets
  de ces entités (personnalisation additive, ADR product-0013).

## Décision — arborescence du dépôt projet

```
<projet>/
├── fabrica.json                  # version de Fabrica épinglée + métadonnées projet
├── entities/
│   └── u_task/
│       ├── entity.json           # attributs (ordre), capacités, traçabilité, display value
│       ├── forms/default.json    # formulaire (MVP : un seul)
│       ├── list-views/default.json
│       ├── value-lists/status.json   # liste qualifiant u_task.status
│       ├── lifecycle.json        # si capacité à-états
│       ├── acl.json              # droits portant sur cette entité
│       ├── scripts/display_value.ts  # un fichier par point d'entrée
│       └── i18n/{en,fr}.json     # libellés entity:/list: de cette entité
├── fabrica-entities/
│   └── sys_user/                 # surcharges projet d'une entité Fabrica (jamais sa définition)
├── security/                     # rôles, définition des groupes (pas leurs membres)
├── i18n/                         # familles ui: et message: (hors entité)
├── config/                       # journalisation, définition des niveaux de traçabilité
├── reports/                      # reporting v1
├── seed/                         # état initial de données (ex. membres initiaux de groupes)
└── .fabrica/                     # généré (types TS, caches) — ignoré par git
```

**Principes de nommage** :
- le **répertoire porte l'identité** (le code d'entité, immuable) ; le **nom de fichier porte le rôle**
  (`entity.json`, `forms/default.json`) ;
- un **script est nommé par son point d'entrée** (`display_value.ts`) : la liaison devient une
  **convention d'emplacement**, pas une référence libre à maintenir ;
- les **listes de valeurs**, nommées par l'attribut qu'elles qualifient (une par attribut — product-0018),
  vivent **dans le répertoire de leur entité** ;
- les **ACL** sont rangées avec l'**entité ciblée** (localité, merges indépendants) ; la vue transverse
  « que donne le rôle X ? » est fournie par Fabrica ;
- répertoires **au pluriel anticipés** (`forms/`, `list-views/`) même avec un seul élément au MVP :
  changer la structure plus tard serait une migration ;
- les **données de production et de test** ne sont **jamais** dans le dépôt (tool-0001).

## Décision — git

- **Git uniquement, aucune dépendance à une forge** : ni API, ni Actions, ni demandes de fusion. Fonctionne
  avec GitHub, Gitea, GitLab ou un dépôt nu.
- **Commits préparés par Fabrica**, avec l'**identité du développeur** authentifié comme auteur (rôle
  unique Développeur au MVP, tool-0002). Un **commit par dette traitée** est une bonne pratique (atomique,
  traçable).
- **Branches, fusions, conflits** : délégués aux outils git standard ; **conflits résolus manuellement**,
  en local.
- **Réservé (post-MVP)** : **pilote de fusion personnalisé** déclaré via `.gitattributes`, fusionnant les
  JSON du métamodèle **sémantiquement** (par élément et identifiant stable). Limite : il ne s'applique
  qu'aux **fusions locales**, pas à celles déclenchées depuis l'interface web d'une forge — cohérent avec
  la résolution locale.

## Décision — validations internes à Fabrica

- **MVP** : les validations dont la **revue de montée de version** et le **packaging** ont besoin —
  valeurs obligatoires manquantes, **intégrité référentielle** (toute entité, attribut, liste référencés
  existent), **complétude des scripts** (tout point d'entrée déclaré a son script). **Internes à
  Fabrica**, exécutées au packaging et **à la demande** depuis l'outil.
- **Post-MVP** : validateur étendu — **contrôle de canonicité** (re-sérialiser et comparer octet à octet,
  reformatage possible), **détection d'orphelins** (script sans point d'entrée déclaré), usage **hors de
  l'outil** (crochets git).
- Une fusion corrompue ne peut pas être *empêchée* au niveau git, mais elle est **détectée avant tout
  usage** — jamais silencieuse.

## Décision — compilation et exécution locale

- **Deux modes de compilation** :
  - **développement** : transpilation à la **sauvegarde**, vérification de types **dans l'éditeur**,
    **rechargement** du métamodèle et des scripts dans l'instance locale — retour immédiat ;
  - **packaging** : compilation complète + vérifications officielles (validations MVP, revue de montée
    de version), qui **font foi**.
- Le **rechargement en développement** (peut interrompre l'instance ou perdre un état) **n'est pas** le
  déploiement à chaud (production sans interruption, ADR product-0014, hors MVP).
- **Exécution locale toujours possible**, **même avec des dettes bloquantes ouvertes** : le développeur
  doit pouvoir tester le traitement de chaque dette, une par une. Les dettes bloquent le **passage en
  RUN** (packaging), **jamais le travail**. Conséquence : l'instance locale **tolère un métamodèle
  incomplet** et se **dégrade proprement** — un attribut obligatoire non renseigné n'empêche pas le
  démarrage ; un script manquant produit une erreur claire sur l'entité concernée, pas un plantage global.
- **MVP** : exécution locale dans **Docker**. Les **tests automatisés écrits par le projet** restent hors
  MVP (au MVP, le développeur teste en utilisant son instance).

## Hors périmètre (renvois / dettes)

- **Alternatives à Docker** pour les postes sans droits d'administrateur (fréquent en entreprise ; licence
  Docker Desktop payante au-delà d'une certaine taille) : conteneurs sans privilèges (Podman rootless),
  environnement de développement distant, exécution native (Node.js et PostgreSQL existent en binaires
  portables). Post-MVP.
- **Ajout d'attributs aux entités système** par le projet (et paramétrage associé de vues, formulaires,
  effets). Post-MVP.
- **Pilote de fusion sémantique**, **validateur étendu**, **crochets git**. Post-MVP.
- **Tests automatisés projet** ; **développement concurrent** complet ; **rôles de l'outil de
  développement**. Post-MVP.
- **Spécification exacte des fichiers** (`format-metamodele.md` + JSON Schema) : tool-0001.

## Alternatives écartées

- **Fichier de script créé par le développeur** : aucune garantie de liaison. Création par Fabrica.
- **Code inline dans le JSON** : dégrade la merge-abilité et prive du typage. Fichier `.ts` séparé.
- **Construire un éditeur de code** : Monaco existe. *Buy*.
- **Fichiers de Fabrica copiés dans le projet** : le projet pourrait les modifier et ils divergeraient.
  Version épinglée, lecture seule.
- **Validations dans la CI d'une forge** : dépendance à une forge et bruit d'un commit par dette.
  Validations internes à Fabrica.
- **Bloquer l'exécution locale sur les dettes bloquantes** : empêcherait de tester chaque correction.
  Seul le packaging est bloqué.
