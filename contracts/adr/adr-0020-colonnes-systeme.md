# ADR-0020 — Colonnes système

**Statut :** Proposé — 2026-07-11
**Portée :** décision du **cœur** (Fabrica), générique (Principe IV). Définit les colonnes présentes
sur **toute** table. À éclater en `contracts/adr/0020-colonnes-systeme.md`.

---

## Contexte

Toute table générée par Fabrica porte un ensemble de **colonnes système** : posées par le système,
jamais écrites par l'utilisateur, présentes partout (homogénéité, réflexion). Elles sont référencées
par les ADR-0012 (hors du modèle de droits utilisateur), 0005 (classe de la ligne), 0010 (auteur via
identité de session).

**Critère d'appartenance à cet ADR** : une colonne système existe sur **toute** table, est **posée
par Fabrica**, est **non modifiable par l'utilisateur**. Ce qui échoue à l'un de ces tests n'est pas
une colonne système — ainsi `number` (n'existe que si l'entité est « numérotée », et il est visible
comme identifiant fonctionnel) **n'est pas** une colonne système : il relève de la **capacité
d'entité « numérotée »** (ADR capacités d'entité à venir), hors périmètre ici.

## Liste normative des colonnes système (toute table)

| Colonne | Rôle |
|---|---|
| `sys_id` | **Identifiant technique**, **UUID**. ID du schéma GraphQL et **valeur des clés étrangères**. Exposé (c'est l'ossature référentielle du graphe) mais **jamais montré à l'utilisateur** dans l'IHM et **jamais une référence fonctionnelle métier** (les humains/processus référencent par l'identifiant fonctionnel, ex. `number`). Immuable. |
| `sys_created_at` | Horodatage de création. |
| `sys_created_by` | Auteur de la création (identité de session, ADR-0010). |
| `sys_updated_at` | Horodatage de la dernière modification. |
| `sys_updated_by` | Auteur de la dernière modification (identité de session). |
| `sys_version` | **Version de la donnée**, incrémentée à chaque modification. Support du **verrouillage optimiste** (ci-dessous). |
| `sys_class` | **Code de l'entité réelle** de la ligne (le nom de la table la plus dérivée). Présente sur **toute** table, y compris sans héritage (homogénéité + réflexion). Sans objet pour une entité sans héritage, mais uniforme. |

## Décision — sys_id

- **UUID**, identifiant purement technique. C'est l'**ID du schéma** et la **valeur des FK** : c'est
  son rôle, il est donc pleinement exposé dans le graphe (relations, cache, liens).
- **Jamais montré à l'utilisateur** dans l'IHM ; **jamais une référence fonctionnelle** — aucun
  processus métier ne désigne un objet par son UUID (on utilise l'identifiant fonctionnel, `number`,
  quand l'entité en a un). Cette invisibilité fonctionnelle préserve la nature **interne et
  remplaçable** du sys_id.
- **Récupérable par l'administrateur** (via les fonctions d'introspection — ADR fonctions
  d'administration à venir).

## Décision — sys_version et verrouillage optimiste

- `sys_version` est incrémentée par Fabrica à chaque modification.
- **Verrouillage optimiste imposé, sur tous les canaux (IHM incluse)** : tout `update` porte sur le
  couple **(sys_id, sys_version)** ; si la version fournie est périmée (un autre a modifié entre-temps),
  l'update est **refusé**. Empêche l'écrasement concurrent silencieux — un référentiel multi-utilisateur
  ne doit jamais perdre une modification sans le dire.
- Adouci à terme par la **collaboration temps réel** (voir renvois) : voir la nouvelle version en direct
  réduit les collisions au lieu de seulement les refuser.

## Décision — sys_class

- Contient le **code de l'entité réelle** (nom de la table la plus dérivée). Sur **toute** table, pour
  l'homogénéité et la **réflexion** (un objet, un écran, un script peut connaître la classe d'une ligne
  uniformément).
- **Sa modification déclenche une re-classification** (déplacer la ligne conformément au modèle
  d'héritage, ADR-0005 — ex. reformater un serveur Linux en Windows). Ce mécanisme (transitions de
  classe autorisées, migration des attributs spécifiques, atomicité multi-tables) est **lourd** et
  **renvoyé à un ADR dédié** ; **pas un prérequis v1**. Ici, on déclare seulement l'existence et le
  sens de la colonne.

## Décision — mécanisme d'écriture

Les colonnes système sont posées par les **crochets `before-create` / `before-update`** du moteur
(ADR-0016), là où l'**identité de session** est connue (ADR-0010) — d'où `sys_created_by` /
`sys_updated_by`. Fabrica ne construit pas de mécanisme spécial : elle réutilise ses propres crochets.

- **Infalsifiables à travers l'application** : aucun utilisateur, script ou canal ne peut poser une
  valeur système mensongère (un `sys_updated_by` falsifié, par exemple).
- **Accès base direct hors garantie** : un accès direct à PostgreSQL peut corrompre ces colonnes.
  **Assumé** (risque d'exploitation connu, ADR-0008) — la garantie porte sur l'application, pas contre
  l'accès base.

## Décision — visibilité et droits

- Les colonnes système sont **lisibles** seulement si l'entité l'est (ADR-0012).
- Elles sont **hors du modèle de droits utilisateur** en écriture : ce n'est pas un « droit refusé »,
  c'est le système qui les pose (ADR-0012).
- Elles portent le préfixe **`sys_`** (préfixe réservé Fabrica, ADR-0017).

## Hors périmètre (renvois)

- **`number`** (identifiant fonctionnel) : **capacité d'entité « numérotée »** — n'existe que si
  l'entité l'active ; alors **obligatoire et unique** ; préfixe (trigramme/quadrigramme, immuable au
  démarrage, changement = migration réservée) + nombre de chiffres (6 par défaut) ; croissant, unique,
  **sans garantie de continuité** (un numéro réservé puis non utilisé laisse un trou — assumé, cf. cas
  du ticket ouvert puis abandonné) ; compteur du prochain numéro stocké par Fabrica. → ADR capacités
  d'entité / numérotation.
- **Capacités d'entité** : mécanisme générique par lequel une entité active un paquet cohérent
  (colonnes, contraintes, comportements) — `à-états` (ADR-0019), `numérotée`, et à venir. → ADR
  distinct ; amendera l'anatomie (ADR-0013).
- **Re-classification** (changement de `sys_class`) : ADR dédié, non prérequis v1.
- **Fonctions d'administration / introspection** (récupérer sys_id, code technique d'un champ,
  représentation YAML/GraphQL d'un objet réutilisable en requête, liste contraignant un champ…) : ADR
  distinct, s'inspirer des fonctions réservées de ServiceNow.
- **Collaboration temps réel** (voir en direct la modification d'un autre : subscription GraphQL ou
  bus, via websockets avec repli) : ADR distinct ; adoucit le verrouillage optimiste.
- **Forme des URLs** : lisibilité/partageabilité (marquer une liste, une liste filtrée, un objet) vs
  révélation de structure (ADR-0009). Tension à trancher ; posture de principe susceptible de céder
  devant l'usage. ADR distinct.

## Alternatives écartées

- **`number` comme colonne système** : n'existe pas sur toute table et est visible/fonctionnel. C'est
  un attribut standard d'une capacité d'entité, pas une colonne système.
- **sys_id comme référence fonctionnelle** (montré, cité par les humains) : le figerait comme
  identifiant métier et lui ôterait sa nature technique remplaçable. Réservé au technique ; le
  fonctionnel est `number`.
- **Verrouillage optimiste optionnel** : laisserait des écrasements concurrents silencieux. Imposé,
  tous canaux.
