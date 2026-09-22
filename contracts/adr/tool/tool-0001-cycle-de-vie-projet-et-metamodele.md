# TOOL-0001 — Cycle de vie d'un projet Fabrica, auto-application et source de vérité du métamodèle

**Statut :** Proposé — 2026-07-11
**Famille :** **Outillage** (`tool/`) — décisions sur *comment on fabrique, package, déploie et
exploite* un projet Fabrica (distinctes des ADR **Produit**, qui portent sur le comportement de
Fabrica et de l'application générée). Générique (Principe IV).

---

## Contexte

Les ADR Produit ont posé *ce que Fabrica fait avec les données*. Cet ADR ouvre la famille Outillage :
le **cycle de vie d'un projet** (de la conception au RUN) et l'outillage que Fabrica fournit pour
chaque phase. Il tranche notamment la **source de vérité du métamodèle** et le **format** de son
stockage — fondateurs, versionnés, contractuels.

## Décision — les phases d'un projet (et leur statut MVP)

1. **Conception** — éditer le métamodèle. **MVP.**
2. **Réalisation / génération** — Fabrica génère DDL PostgreSQL + schéma GraphQL + IHM. **MVP.**
3. **Test** — Fabrica pose et exécute les tests qu'elle **sait** définir (invariants du généré) ; les
   tests *projet* (logique custom) sont **hors MVP**. **MVP (minimal).**
4. **Packaging** — produire l'artefact déployable (image, ADR Produit-0013) ; c'est ici que les
   **dettes de migration** bloquent si non traitées (ADR tool à venir). **MVP.**
5. **Déploiement** — déployer sur une instance. **MVP.**
6. **Exécution (RUN)** — l'application tourne. **MVP.**
7. **Exploitation avancée** (sauvegarde, clonage d'environnement, montée de version à chaud, haute
   disponibilité, développement concurrent) — **hors MVP** (stubs / à cadrer). MVP exploitation =
   seulement l'accès administrateur aux logs.

## Décision — auto-application : Fabrica édite son propre métamodèle

Fabrica **s'édite elle-même** : le métamodèle est un métamodèle-client, édité **avec les écrans que
Fabrica génère**. Il n'y a **pas d'éditeur de métamodèle séparé à construire** — Fabrica appliquée à
son propre métamodèle *est* l'éditeur. Cohérent avec le métamodèle réflexif (ADR Produit-0019).

## Décision — bootstrap (méta-métamodèle)

L'auto-application exige un **noyau bootstrap** : le **méta-métamodèle** (la description de « ce
qu'est un métamodèle » : une entité a des attributs, un attribut a un type, une relation…) est
**posé à la main une première fois** (en git), à partir duquel Fabrica génère les écrans qui
l'éditent — comme un compilateur qui se compile lui-même. Une fois amorcé, tout (y compris le
méta-métamodèle) s'édite avec les écrans Fabrica. Le méta-métamodèle utilise les **préfixes réservés
Fabrica** (pas `u_`, réservé aux entités **projet**). *(Une v0 du méta-métamodèle sera proposée.)*

## Décision — source de vérité : git

- **Git est la source de vérité unique** du métamodèle (Principe I : le commit est l'acte de
  décider ; c'est ce qui est packagé et déployé).
- L'éditeur (Fabrica auto-appliquée) **charge depuis git**, édite via un **support de travail libre**
  (mémoire, base temporaire… — non imposé), et **réécrit vers git**. Pas de base persistante
  intermédiaire faisant foi.
- **Enregistrement au fil de l'eau, asynchrone**, résistant à la corruption (crash Fabrica/PC) →
  **plusieurs petits fichiers** plutôt qu'un monolithe (le rayon de corruption est borné, l'écriture
  incrémentale triviale).

## Décision — frontière : métamodèle / seed / données de production

| Nature | Contenu | Où |
|---|---|---|
| **Métamodèle** | structure (entités, attributs, relations, héritage), **listes de valeurs**, comportement (règles, vues, cycles de vie), gouvernance-définition (rôles, ACL, définition des groupes, axes de rattachement), traductions (hors `system`), config (gabarits appliqués) | **git** ; **immuable en prod** |
| **Seed** | état **initial** de *données* versionné pour le déploiement (ex. **membres initiaux d'un groupe**) — c'est de la donnée, mais dont on fixe un point de départ livré | **git** ; **joué au déploiement**, puis évolue en prod |
| **Données de production** | données métier nées et modifiées **en prod** (et données de test — même nature) ; appartenances de groupes en RUN | **instance**, **jamais git** (le terme « production » rappelle la **confidentialité** associée) |

- Critère : *défini par le concepteur, immuable en prod* → métamodèle (git) ; *état initial livré
  d'une donnée qui évoluera* → seed (git) ; *né/modifié en prod* → données de production (instance).
- **Une liste de valeurs ne vit qu'en git** (ne change jamais en prod — sinon on casserait le contrat
  d'API, ADR Produit-0018). Ce n'est **pas** du seed.
- **Références** : vers une **liste de valeurs** = par **code** (`codeEntité.codeÉlément`,
  versionné-stable) ; vers un **objet** = par **sys_id** (UUID, donnée volatile). Deux natures de
  contrainte référentielle (ADR Produit-0018/0023).

## Décision — format des fichiers de métamodèle

Le métamodèle est une **arborescence de fichiers JSON** (pas un fichier) : **un fichier par entité**,
un par liste, etc. — découpage qui sert la robustesse (fil de l'eau), le merge et la lisibilité.

**Propriétés fondatrices (dans cet ADR, ne bougent pas) :**
- **JSON**, formaté **canoniquement, une propriété par ligne**, **ordre des propriétés déterministe**,
  indentation fixe. → deux modifications de deux champs d'un même objet sont sur **deux lignes** →
  **merge sans conflit** (contrairement au tabulaire, où deux champs d'un enregistrement sont sur la
  même ligne). C'est **la merge-abilité** — exigence-mère (fusion transparente de plusieurs
  développeurs) — qui commande ce choix, pas la seule lisibilité.
- **Idempotence** : `lire(écrire(m)) == m` ; même sémantique ⇒ **mêmes octets** (pas de faux diff).
- **Identité stable des éléments** : chaque élément de métamodèle (entité, attribut, règle) porte un
  **identifiant stable** — pas sa position ni son nom (qui peut changer). Sans quoi « renommer »
  apparaît comme « supprimer + ajouter », et deux renommages concurrents deviennent ingérables.
- **Validation post-merge** : après tout merge git, Fabrica **re-valide la cohérence** du métamodèle
  fusionné avant de l'accepter — un merge textuel réussi peut produire un métamodèle invalide.

**Spécification technique (artefacts séparés, `contracts/`) :**
- `contracts/format-metamodele.md` : documentaire (organisation des fichiers, découpage, nommage,
  exemples).
- `contracts/format-metamodele.schema.json` : **JSON Schema** validant les fichiers (une famille de
  schémas — un par type de fichier : entité, liste…). Largement **dérivé du méta-métamodèle**.

**Le format est un contrat public versionné** (constitution, « Contrat public ») : il évolue en
**compatibilité ascendante** (ajout de champ optionnel = additif ; passage obligatoire ou retrait =
migration). La **politique de montée de version du format** (quand un champ peut devenir obligatoire ;
échec de l'application du patch **en environnement de dev, avant packaging**, si le champ n'est pas
rempli partout) relève de l'**ADR dettes de migration** (tool à venir), pas d'ici.

## Décision — diff sémantique (confort de lecture)

Fabrica fournit (MVP minimal) un **diff sémantique** du métamodèle (« l'attribut X a été ajouté à
l'entité Y ») — plus lisible qu'un diff textuel. Il **aide à lire/arbitrer**, mais **ne remplace pas**
le merge git (qui opère sur le texte brut, avant Fabrica) : la merge-abilité vient du **format**, pas
du diff sémantique.

## Hors périmètre (renvois / dettes — famille tool)

- **Dettes de migration / gate de packaging** (dont la politique de montée de version du format ;
  tableau de bord de dettes plutôt qu'un wizard ; blocage du passage en RUN tant que non traité).
- **Développement concurrent** complet (résolution assistée de conflits) ; **affichage des écarts**
  entre versions/branches/tags (use cases de gestion de conf).
- **Droits dans l'outil de développement** : *second* système d'autorisation (qui peut modifier le
  métamodèle), **distinct** de l'autorisation du RUN (ADR Produit-0012).
- **Définition de tests par le projet** ; **haute disponibilité** (architecture générée) ;
  **exploitation** avancée (sauvegarde, clonage, monitoring…).
- **Déploiement à chaud** (ADR Produit-0014) : hors MVP (pendant le MVP, on est dans la fenêtre de
  régénération — incompatibilités acceptées, cf. ci-dessous).

## Note — le MVP est dans la fenêtre de régénération

Pendant le MVP, le cœur reste un **spike** (ADR Produit-0002) : des montées de version de Fabrica
peuvent **casser** le périmètre projet, et c'est **accepté** — le MVP sert aussi à **tester la
robustesse** du paramétrage projet face aux évolutions de Fabrica. La bascule « cœur figé /
compatibilité ascendante garantie » intervient après.
