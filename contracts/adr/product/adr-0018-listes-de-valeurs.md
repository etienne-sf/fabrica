# ADR-0018 — Listes de valeurs : structure, désactivation, conditionnement

**Statut :** Proposé — 2026-07-11
**Portée :** décision du **cœur** (Fabrica), générique (Principe IV). Mécanisme neutre de base ; le
cycle de vie, la projection API et la surcharge/modules sont renvoyés à leurs ADR. À éclater en
`contracts/adr/0018-listes-de-valeurs.md`.

---

## Contexte

Une **liste de valeurs** conditionne un attribut dans un **domaine de valeurs** (ex. le statut d'une
tâche parmi un ensemble défini). À distinguer nettement des **données d'une entité** (les
enregistrements métier) — hors sujet. Une liste de valeurs est un **mécanisme neutre unique** : il
n'y a pas plusieurs « natures » de listes ; ce que le cycle de vie ajoute (transitions, contrôles)
se greffe *par-dessus*, ailleurs (ADR cycle de vie à venir).

## Décision — structure

- Une liste de valeurs est une **table de référence**, **jamais un `enum`** : éditable à chaud sans
  migration (décision fondatrice, cohérente ADR-0004 et déploiement à chaud ADR-0014).
- Une **valeur** porte au démarrage : **code** (identifiant technique stable), **label** (traduit —
  ADR-0017), **ordre** d'affichage, **flag actif/inactif**. Pas de couleur ni d'icône (réservé).
- Une liste est **nommée par l'attribut qu'elle qualifie** (`u_task.status`), **une liste par
  attribut**, duplication assumée si deux attributs veulent les mêmes valeurs (elles divergent
  souvent — ADR-0017).

## Décision — stockage par référence (FK)

La valeur choisie pour un attribut est stockée comme **référence (clé étrangère) vers la table des
valeurs de la liste**. Justification :
- **Intégrité garantie en base** : impossible de stocker une valeur hors-liste (cohérent « invariants
  en base », ADR-0004).
- **Garde-fou anti-suppression** : la FK empêche de supprimer une valeur référencée, ce qui **impose
  la désactivation** plutôt que la suppression (ci-dessous). La FK *sert* la politique, elle ne la
  gêne pas — la désactivation laisse la ligne en place, la FK reste valide.

## Décision — désactivation, jamais suppression

- « Retirer » une valeur = la **désactiver** (flag actif=faux), jamais l'effacer (cohérent
  append-only). Une valeur inactive **survit dans les données existantes** mais **ne peut plus être
  écrite**.
- **Contrôle de conformité uniquement sur les attributs effectivement modifiés** (comparaison
  entrant/existant — même principe que l'autorisation par attribut, ADR-0012). Un objet portant une
  valeur inactive dans un champ **qu'on ne modifie pas** est laissé tel quel. On ne contrôle que ce
  qui est touché.
- Conséquences de ce choix : pas de migration de masse des objets « en retard » ; et le cas du
  **champ caché/non modifié se règle de lui-même** (non modifié = non contrôlé). Un utilisateur n'est
  jamais coincé à devoir corriger un champ qu'il ne touchait pas.

## Décision — conditionnement d'une liste par un autre attribut

Le domaine de valeurs d'un attribut peut être **conditionné par la valeur d'un (seul) autre
attribut** (ex. liste fille filtrée selon une liste mère ; valeurs proposées selon une case à
cocher).
- **Limité à UN attribut conditionnant** (pas de combinaison de plusieurs) : couvre le cas courant
  sans ouvrir la complexité combinatoire.
- **Récursif en chaîne** : A conditionne B, B conditionne C… sans limite de niveaux. Récursif en
  profondeur, simple en largeur (chaque maillon ne dépend que d'un attribut).
- **Déclaré dans la définition de la liste** (c'est un attribut de la liste : « mes valeurs dépendent
  de l'attribut X, selon telle correspondance ») — d'où sa place dans cet ADR.
- **Appliqué via le moteur des effets** (ADR-0006 / ADR-0016) : le filtrage des valeurs proposées et
  sa déclinaison multi-canal réutilisent le mécanisme d'effets, ils ne le réinventent pas. La liste
  *déclare* la contrainte, les effets l'*exécutent*.

## Hors périmètre (renvois)

- **Cycle de vie des objets** (états = une liste, plus transitions et contrôles) → ADR à venir.
- **Conditionnement par plusieurs attributs** : écarté (limité à un attribut).
- **Projection des listes dans le contrat d'API** (type énuméré exposé, impact des montées de
  version sur les consommateurs) → ADR à venir (délicat à cause du versionnement).
- **Surcharge / extension des listes standards** (quand un projet réutilise et étend une table d'un
  module) → lié à la notion de **module**, réservé.

## Alternatives écartées

- **`enum`** : non éditable à chaud (migration à chaque changement). Remplacé par table de référence.
- **Stockage du code sans FK** : perd l'intégrité en base et le garde-fou anti-suppression. La FK est
  retenue.
- **Suppression d'une valeur** : casse les données existantes qui la référencent. Remplacée par
  désactivation.
- **Contrôle de conformité sur tous les attributs** (pas seulement les modifiés) : coincerait
  l'utilisateur sur des champs non touchés / cachés. Limité aux attributs modifiés.
- **Conditionnement par plusieurs attributs** : complexité combinatoire non justifiée au démarrage.

## Conséquences

- Le conditionnement crée un couplage liste↔attribut à appliquer par le moteur des effets : quand la
  valeur conditionnante change, le domaine de l'attribut conditionné est recalculé (cf. ADR effets —
  y compris le sort d'une valeur devenue incompatible).
- La liste étant une table, elle bénéficie de l'édition à chaud (ADR-0014) et de l'i18n de ses
  valeurs (ADR-0017).
