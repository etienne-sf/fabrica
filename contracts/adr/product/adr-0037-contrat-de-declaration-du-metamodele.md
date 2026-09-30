# ADR-0037 — Contrat de déclaration du métamodèle : entités, attributs, types

**Statut :** Proposé — 2026-09-29
**Portée :** décision du **cœur** (Fabrica), générique (Principe IV). **Contrat central** annoncé par
l'ADR-0003 et relevé comme trou n°1 par la revue du 2026-09-26. Le méta-métamodèle v0 et la
spécification du format (tool-0001) en découlent. Le **langage de conditions** fait l'objet d'un ADR
distinct (à venir, après étude des alternatives).

---

## Contexte

Les propriétés d'une entité et d'un attribut étaient dispersées dans une vingtaine d'ADR, et plusieurs
manquaient : types, contraintes, unicité, valeurs par défaut, représentation JSON, nullité. Cet ADR les
rassemble et les complète. Exigence transverse : **toute la conception prévoit une future exposition
JSON**, même si elle n'est pas au MVP — chaque type a une représentation JSON définie.

## Décision — identité : le code, immuable dès le commit

- L'identité d'une **entité** est son **code** ; celle d'un **attribut** est le couple **(code entité,
  code attribut)**. Il n'y a pas d'identifiant technique supplémentaire : le code, déjà utilisé comme
  nom de répertoire (tool-0003), racine des clés de traduction (ADR-0017), dans les URL (ADR-0034) et le
  schéma GraphQL (ADR-0011), **est** l'identité stable exigée par tool-0001.
- **Renommer un code est trop impactant** (cela casse toute interface connectée à un projet). Un code est
  donc **renommable tant qu'il n'est pas commité**, et **immuable ensuite**. Fabrica sait ce qui est
  commité en comparant l'arbre de travail avec le dernier commit (`HEAD`) — elle prépare déjà les
  commits (tool-0003).
- Un renommage avant commit est un **refactoring** : Fabrica met à jour **en même temps** tout ce qui
  cite le code (conditions, rapports, vues, ACL, références d'autres entités, clés de traduction).
- Après commit : renommage **interdit au MVP** ; plus tard, opération explicite avec migration.

## Décision — propriétés d'une entité

| Propriété | Contenu | Source |
|---|---|---|
| code | préfixé (`u_` pour le projet ; préfixes réservés pour Fabrica) | ADR-0017 |
| libellé | **un** libellé, traduit dans plusieurs langues ; description facultative | ADR-0017 |
| parent | entité mère éventuelle (héritage par table de classe) | ADR-0005 |
| capacités | capacités optionnelles activées, avec leur paramétrage | ADR-0025 |
| niveau de traçabilité | niveau par défaut des attributs | ADR-0028 |
| valeur d'affichage | attribut désigné ou attribut calculé | ADR-0032 |
| ordre des attributs | ordre de la liste des attributs, défaut de tous les usages | tool-0001 |

Les capacités système (colonnes système, `actif`, valeur d'affichage, audit) sont présentes sur toute
entité et ne se déclarent pas (ADR-0025).

## Décision — propriétés d'un attribut

| Propriété | Applicable à | Remarque |
|---|---|---|
| code | tous | identité avec le code de l'entité |
| libellé (+ description) | tous | un libellé traduit ; description facultative |
| type, sous-type | tous | voir catalogue ci-dessous |
| obligatoire | tous | vrai/faux ; **absolu** : les effets ne peuvent que resserrer (ADR-0006) |
| unique | tous sauf `text/rich` | vrai/faux ; voir « Unicité » |
| longueur maximale | `text` | `rich` : 10 ko par défaut (contenu stocké, balisage compris) |
| format | `text/line` | catalogue fermé de formats |
| précision, échelle | `number/decimal` | |
| minimum, maximum | `number`, `temporal` | |
| valeur par défaut | tous | constante, ou dynamique (voir ci-dessous) |
| entité cible | `reference` | peut être une entité mère (référence polymorphe) |
| liste de valeurs | `valuelist` | la liste qualifiant l'attribut (ADR-0018) |
| calculé | tous (texte au MVP) | par script, matérialisé (ADR-0032) |
| historisé | tous | ADR-0027 |
| niveau de traçabilité | tous | surcharge celui de l'entité (ADR-0028) |

- **Pas de composant de rendu dans l'attribut.** Ce qui change la donnée (retours à la ligne, balisage)
  est porté par le sous-type ; le widget se **déduit** du type et du sous-type (ADR-0030). Choisir un
  autre affichage est une propriété de la **vue** (le même attribut peut s'afficher autrement dans deux
  formulaires) : surcharge dans la définition du formulaire, **post-MVP**.
- **Jamais multivalué** (constitution).

## Décision — types et sous-types

Les types sont regroupés par **nature de la donnée** ; le **sous-type** fixe le stockage, la
représentation JSON, le scalaire GraphQL et les contraintes. (Regrouper par type JSON serait une
impasse : en JSON, une date est une chaîne, et un décimal doit l'être aussi pour ne pas perdre de
précision.)

| Type | Sous-type | JSON | Scalaire GraphQL | Spécification (`@specifiedBy`) |
|---|---|---|---|---|
| `text` | `line` | chaîne | `String` | intégré |
| `text` | `multiline` | chaîne | `String` | intégré |
| `text` | `rich` | chaîne (balisage nettoyé) | `String` | intégré |
| `number` | `integer` | nombre (32 bits) | `Int` | intégré |
| `number` | `decimal` | **chaîne** | `Decimal` | chillicream/decimal — **à vérifier** |
| `boolean` | — | booléen | `Boolean` | intégré |
| `temporal` | `date` | chaîne `YYYY-MM-DD` | `LocalDate` | andimarek/local-date |
| `temporal` | `time` | chaîne ISO 8601 | `LocalTime` | chillicream/local-time |
| `temporal` | `datetime` | chaîne RFC 3339, en UTC | `DateTime` | andimarek/date-time |
| `reference` | — | chaîne (`sys_id`) | `ID` + objet navigable | intégré |
| `valuelist` | — | chaîne (code) | `String` | intégré |

- **`text` : un type, trois sous-types** (homogénéité). Chaque sous-type porte ses règles : `rich` exige
  un **nettoyage côté serveur** du balisage (liste blanche — protection contre le XSS stocké). Changer
  de sous-type est une évolution du schéma, avec ses interdits (`rich` → `line` perdrait de
  l'information).
- **`datetime`** : stocké en UTC, affiché dans le fuseau de l'utilisateur (préférences, ADR-0017).
  **`date`** et **`time`** n'ont pas de fuseau.
- **`integer`** : borné à 32 bits au MVP (limite du `Int` GraphQL) ; grands entiers (`long`) post-MVP.
- **`decimal`** : jamais de nombre à virgule flottante. Avant d'adopter la spécification ChilliCream,
  vérifier qu'elle sérialise en **chaîne** ; sinon, en retenir une autre.

## Décision — scalaires : spécifications de référence, implémentations existantes

- Tout scalaire personnalisé **référence sa spécification** par `@specifiedBy`, de préférence dans
  l'annuaire de la GraphQL Foundation (`scalars.graphql.org`, spécifications immuables), à défaut une RFC.
- **Aucune implémentation de scalaire n'est développée dans Fabrica** : on utilise une bibliothèque
  existante (candidate : `graphql-scalars`, The Guild). Choix précis en phase technique, sur deux
  critères : **conformité à la spécification référencée**, **présence de `@specifiedBy`** (à défaut,
  Fabrica ajoute la directive au schéma généré, sans réimplémenter le scalaire).

## Décision — formats de chaîne

Catalogue **fermé**, aligné sur le vocabulaire standard du mot-clé `format` de **JSON Schema** (ce qui
rend la génération future d'une spécification JSON directe). Chaque format est associé à un scalaire
GraphQL et à sa spécification. MVP : `email` (RFC 5322), `uri` (chillicream/uri), `uuid`
(chillicream/uuid). Le vocabulaire s'étend par Fabrica ; l'expression régulière libre (mot-clé
`pattern`) est post-MVP.

## Décision — valeurs par défaut

Une **constante**, ou une **valeur dynamique** d'un petit catalogue : utilisateur courant, date ou
date-heure de création. Appliquée **à la création seulement**.

## Décision — unicité

- **Absolue** : unique parmi **tous** les objets, actifs ou non (un objet inactif peut redevenir actif).
- Sur **un attribut seul** au MVP, déclarée par le projet pour ses entités.
- Comparaison **exacte** au MVP. Post-MVP : unicité sur plusieurs attributs, insensibilité à la casse,
  unicité parmi les seuls objets actifs.

## Décision — héritage

Une entité fille **hérite de tous les attributs** de sa mère ; au MVP, elle **ne peut pas modifier**
leurs propriétés. Un code d'attribut est **unique dans toute la hiérarchie** : une fille ne peut pas
redéclarer un code de sa mère. L'identité d'un attribut hérité est celle de sa déclaration (code de
l'entité qui le déclare).

## Décision — droits et nullité

- **Obligatoire ⇒ non nul, partout** : en entrée (création, modification) **et** en sortie.
- **Jamais de valeur remplacée par nul.** Une requête qui **lit, filtre ou trie** un attribut que
  l'utilisateur n'a pas le droit de lire est **rejetée entière, à l'analyse**, avant exécution, avec une
  erreur qui nomme l'attribut (pas de fuite : le schéma est public). Un rejet pendant l'exécution
  produirait des réponses partielles, les nuls remontant jusqu'aux objets parents.
- Conséquence pour l'interface : l'interprète (ADR-0030) construit ses requêtes à partir du
  métamodèle **filtré par les droits effectifs** de l'utilisateur.
- **Lignes invisibles** (filtrage par ligne, ADR-0026) : simplement **absentes** des listes — ce n'est
  pas une erreur.
- **Référence vers une ligne invisible** : la navigation renvoie un **objet restreint** — le `sys_id`
  de la cible et une indication « non accessible », **rien d'autre** (ni attribut, ni valeur
  d'affichage). Non nul, sans fuite de contenu. Vaut aussi pour l'affichage de l'**historique**
  (ADR-0027) et des champs Référence.
- **Ajouter un attribut obligatoire** à une entité peuplée exige une valeur pour les lignes existantes :
  impact **bloquant** de la revue de montée de version (tool-0002) ; règles d'évolution du schéma à
  définir (trou n°3 de la revue).

## Décision — rangement

Entité et attributs dans `entity.json` ; listes dans `value-lists/` ; **effets et leurs conditions dans
`effects.json`** (ajouté à l'arborescence de tool-0003) ; autres fichiers inchangés.

## Hors périmètre (renvois)

- **Langage de conditions** : ADR dédié, après étude des alternatives (JSON Logic, CEL, JSONata,
  filtres type MongoDB ou Hasura, `$filter` OData, RSQL, filtres Frappe, domaines Odoo). Critère :
  grammaire unique, traduisible en SQL **et** évaluable par le moteur de règles, sérialisée en JSON
  fusionnable.
- **Évolution du schéma projet** (ajout, retrait, changement de type ou de sous-type).
- Montants avec devise, durées, fichiers et pièces jointes, grands entiers, `pattern`.
- Surcharge du widget dans les vues ; surcharge des propriétés héritées.

## Alternatives écartées

- **Identifiant technique séparé et code renommable** : un renommage se propagerait partout et
  casserait les consommateurs de l'API.
- **Types calqués sur les types JSON** : dates et décimaux deviendraient des chaînes.
- **`richtext` comme type distinct** de `string` : incohérent (deux types de chaînes, un troisième à
  part). Un type `text`, trois sous-types.
- **Composant de rendu dans l'attribut** : mélange donnée et présentation.
- **Valeur inaccessible remplacée par nul** : typage mensonger, erreurs silencieuses.
- **Unicité parmi les seuls objets actifs** au MVP : conflits à la réactivation.
