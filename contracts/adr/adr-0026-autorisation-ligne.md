# ADR-0026 — Autorisation par ligne (RLS)

**Statut :** Proposé — 2026-07-11
**Portée :** décision du **cœur** (Fabrica), générique (Principe IV). Réalise l'**axe ligne**
d'autorisation (ADR-0012) via la **RLS réservée** de l'ADR-0010. À éclater en
`contracts/adr/0026-autorisation-ligne.md`.

---

## Contexte

L'ADR-0012 pose deux axes d'autorisation (structurel entité/attribut, et **ligne**), pilotés par
les mêmes rôles, avec l'**invariant de conjonction** (le droit effectif = structurel ET ligne ; la
ligne ne peut que *restreindre*). Cet ADR définit l'axe ligne : une ACL dont la **cible est une
condition sur les lignes**. Le mécanisme est la **RLS PostgreSQL**, dont le câblage est déjà posé,
inerte (prédicat `TRUE`), et alimenté par l'identité de session (`SET LOCAL app.user_id`, ADR-0010).

## Décision — l'axe ligne est une ACL à cible « condition »

Une ACL de ligne = **(rôle, opération, cible=condition, )** : « ce rôle accorde tel niveau
(`lire|écrire|supprimer`) sur les lignes qui **satisfont telle condition** ». La condition est
générée en **prédicat RLS** SQL par Fabrica. Invariant repris de l'ADR-0012 : l'axe ligne ne fait
que **restreindre** l'axe structurel (conjonction) — jamais élargir.

## Décision — axes de gouvernance (catalogue fermé)

Une condition de ligne se fonde sur un **axe de gouvernance** : la dimension par laquelle
l'utilisateur courant est relié à la ligne. Catalogue **fermé** (constitution § Catalogues) :
- **utilisateur** : la ligne est reliée à l'identité courante (ex. propriétaire/créateur).
- **groupe** : la ligne est reliée à un des groupes de l'utilisateur.

(Le **rôle** comme axe est écarté au démarrage — sans valeur distincte du groupe.) Chaque axe sait
résoudre ses valeurs **pour l'utilisateur courant** (mes groupes, mon id), une **fois par requête**,
déposées en contexte de session (ADR-0012, résolution ciblée).

## Décision — accroche déclarée par l'entité

Une entité filtrée par ligne déclare **par quel attribut elle s'accroche à un axe** : un couple
**(attribut de l'entité, axe de gouvernance)**. Ex. « `serveur.responsible_group` s'accroche à
l'axe *groupe* ». Fabrica génère le prédicat RLS « la valeur de cet attribut est parmi les valeurs
de l'axe pour l'utilisateur courant ». **Contrat** : Fabrica fournit les axes ; le projet déclare
l'accroche. Le projet **n'écrit pas de SQL**.

## Décision — catalogue des conditions (deux familles)

Catalogue **fermé** de conditions (→ `contracts/catalogues/`), deux familles :
1. **condition sur valeur** : un champ comparé à une constante (« statut = brouillon »).
2. **condition sur relation à l'utilisateur courant** : un champ comparé à une valeur d'axe (« le
   groupe de la ligne est un de mes groupes ») — le **cas dominant** (« je vois mon périmètre »).

## Décision — limite dure : SQL uniquement

Une condition de ligne **infranchissable doit être exprimable en prédicat SQL/RLS**. Une règle qui
exigerait une **fonction JS arbitraire** n'est **pas** un candidat RLS (le sandbox JS ne s'exécute
pas dans un prédicat Postgres — ADR-0016 est *au-dessus* de la base). Hors SQL : soit une **fonction
PostgreSQL** (reste en base, infranchissable, mais SQL/plpgsql), soit un **filtrage de confort
contournable** (jamais une garantie). Le catalogue de conditions n'admet que le SQL-exprimable.

## Décision — performance : jointure d'abord, mesure ensuite

- **Rattachement direct** (la ligne porte l'attribut d'accroche) : le prédicat teste l'appartenance
  à l'ensemble des valeurs d'axe **déjà en session** (pré-calculé une fois par requête) — **pas de
  jointure par ligne**. Efficace, couvre le cas courant.
- **Rattachement indirect** (la ligne hérite de l'accroche d'une entité référencée — ex. l'incident
  via son serveur) : réalisé par **jointure dans le prédicat** au démarrage. **Pas de
  dénormalisation par défaut** (constitution) : on ne matérialise/propage l'accroche qu'**après
  avoir mesuré** un problème sur un cas précis. Le multi-sauts est réservé.
- Les colonnes d'accroche sont **indexables** (anticipé, ADR-0004).

## Cas d'usage rattachés

- **Notes / commentaires** : table de Fabrica portant l'attribut système `sys_class` (l'entité
  commentée). **Démarrage** : filtrées sur les classes lisibles (via l'axe structurel, ADR-0012).
  **Cible à terme** : le commentaire ne doit pas être plus visible que la **ligne** commentée —
  droit dérivé par ligne (jointure vers l'entité cible). Cas dur, cohérent avec cet ADR.
- **Délégation de gestion de groupe** (ADR-0012) : exprimée comme **ACL de ligne sur la table des
  appartenances** (« gérer les membres du groupe G » = droit d'écriture sur les lignes
  d'appartenance où groupe = G).

## Points de vigilance (ADR-0010, déjà notés)

- **Pooler externe** (PgBouncer en mode transaction) peut dissocier `SET LOCAL` de la requête.
- **Résolution bornée** : un utilisateur à très nombreux groupes alourdit le contexte de session.

## Alternatives écartées

- **Rôle comme axe de gouvernance** : sans valeur distincte du groupe au démarrage.
- **Conditions de ligne en script JS** : non exprimables en RLS (limite dure). SQL/fonction Postgres
  uniquement pour l'infranchissable.
- **Dénormalisation systématique du rattachement indirect** : optimisation prématurée
  (constitution). Jointure d'abord, dénormalisation seulement sur mesure.
- **Multi-sauts** : coût de jointures en cascade par ligne. Réservé.

## Renvois

- ADR-0012 (modèle d'ACL, résolution, invariant de conjonction), ADR-0010 (identité/RLS/`SET
  LOCAL`), ADR-0016 (limite JS), ADR-0004 (index).
- **Catalogue des conditions de ligne** à créer dans `contracts/catalogues/`.
