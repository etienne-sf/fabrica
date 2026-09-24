# ADR-0034 — Forme des URLs

**Statut :** Proposé — 2026-07-11
**Famille :** **Produit** (`product/`). Générique (Principe IV).

---

## Contexte

Il faut des URLs **stables** pour adresser objets et listes (partage, marque-page). Tension notée :
**lisibilité/partage** (une URL compréhensible) vs **révélation de structure** (exposer le nom
d'entité). Et un choix d'identifiant : **numéro** (lisible, mais seulement pour les entités
numérotées) ou **sys_id** (UUID, universel).

## Décision — URL par `sys_id` au MVP

- Forme : **`/…/{code_entité}/{sys_id}`** pour un objet ; **`/…/{code_entité}`** pour une liste.
- **`sys_id` uniquement au MVP** — c'est le cas qui **marche toujours** : toute entité a un `sys_id`
  (colonne système universelle, ADR-0020), **numérotée ou non**. Le `number` n'existe que pour les
  entités numérotées ; une URL par numéro ne couvrirait pas tout. Au plus simple : une seule voie,
  universelle.
- Le `sys_id` en URL reste un **identifiant technique de lien** ; il ne devient **pas** une référence
  fonctionnelle métier (l'humain cite le `number` quand il y en a un). **Cohérent avec l'ADR-0020**
  (sys_id technique, présent dans le graphe et les liens, jamais l'identifiant métier manipulé).

## Décision — la sécurité n'est pas l'obscurité de l'URL

Exposer le **`code_entité`** dans l'URL ne crée aucune faille : la **frontière de sécurité est
l'API** (ADR-0008), pas l'obscurité du nom d'entité. Connaître le nom d'une entité ne donne aucun
accès qu'on n'aurait pas déjà. La lisibilité/partageabilité prime donc sans coût de sécurité.

## Hors périmètre (renvois / réservé)

- **URL par `number`** (forme lisible/humaine, partage) : **réservée** (post-MVP). Les deux formes
  cohabiteront : `sys_id` (UUID) et `number` (préfixe+chiffres) ont des **formats disjoints**, donc
  Fabrica pourra résoudre l'un ou l'autre sans ambiguïté ; le `number` serait alors la forme
  **canonique/humaine**, le `sys_id` la forme **technique** (lien direct sans jointure, entités sans
  numéro).
- **Listes filtrées** adressables par URL ; **canonicité web** (une ressource, une URL) : à préciser
  post-MVP.

## Alternatives écartées

- **URL par `number` au MVP** : ne couvre pas les entités non numérotées ; imposerait un repli.
  `sys_id` universel au MVP.
- **URL opaque** (`/o/{id}` sans entité) : perd la lisibilité sans gain de sécurité (la sécurité est
  l'API). On expose le `code_entité`.
