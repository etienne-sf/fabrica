# ADR-0023 — Relations entre entités

**Statut :** Proposé — 2026-07-11
**Portée :** décision du **cœur** (Fabrica), générique (Principe IV). Mécanisme fondamental dont
dépendent : `sys_created_by`/`sys_updated_by` (ADR-0020), les FK d'héritage (ADR-0005, cas interne),
l'affectation, les liens inter-entités. À éclater en `contracts/adr/0023-relations-entre-entites.md`.

---

## Contexte

Une **relation** entre deux entités se matérialise par une **clé étrangère** portant le `sys_id`
(UUID) de l'objet cible (ADR-0020). On a supposé « une FK » un peu partout sans poser le mécanisme :
cet ADR le définit, pour la relation **déclarée par le projet** entre entités métier. (Les FK
*internes* — héritage ADR-0005, listes de valeurs ADR-0018 — sont des usages distincts, traités
ailleurs.)

## Décision — relation vs liste de valeurs : le critère

Référencer par **code lisible** (liste de valeurs) ou par **identité technique** (relation) dépend
de la **volatilité du domaine et de l'acteur en charge** :
- **Liste de valeurs** : domaine **borné et stable**, modifié par un **acte d'administration**. On
  peut **imposer à l'administrateur** un code technique lisible (court, représentatif, unique dans la
  liste, en anglais). Le référencer reste lisible (`status: "closed"`).
- **Relation** : cible **volatile, non bornée**, créée en masse par des **utilisateurs standards**
  (incidents, CI, utilisateurs). On **ne peut pas** imposer un code lisible → on référence par
  **sys_id (UUID)**, non lisible.
- **Cas limites** (liste volatile, nombreux éléments) : **choix de l'utilisateur** — liste ou entité.
  Les conséquences (dont la perte de lisibilité des références) doivent être **documentées dans
  l'interface d'administration**.
- Conséquence assumée : un export/JSON n'est **pleinement lisible que pour les listes** ; les
  relations vers objets volatils y apparaissent en UUID.

## Décision — une relation, deux points de vue

Une relation est **unique** et **bidirectionnelle**, matérialisée par **une seule FK** (portée par le
côté « plusieurs »), **lue dans les deux sens** :
- **Côté « 1 » (porteur de la FK)** : c'est un **attribut de type référence**, avec entité cible. Il
  vient avec son code, son nom, sa description et leurs traductions — **déjà couvert** (attribut
  standard, ADR-0003/0017). Ex. `incident.requester → utilisateur`.
- **Côté « N » (navigation inverse)** : « les incidents de cet utilisateur ». Son libellé (l'onglet
  du formulaire) est un **texte de traduction** (famille `ui`/`entity`, ADR-0017) — pas un nouvel
  attribut.
- Fabrica **génère les deux sens de navigation** dans le schéma GraphQL (ADR-0011) à partir de la
  seule relation déclarée.

**Affichage** : montrer ou non l'onglet inverse (côté N) dans un formulaire est une **propriété de la
vue**, pas de la relation (il peut y avoir plusieurs vues d'un objet ; une vue admin par défaut
affiche toutes les relations — attributs côté 1 et onglets côté N). Cohérent contrat d'IHM (ADR-0003).

## Décision — cardinalités

- **N:1 / 1:N** : gérées par Fabrica (une FK, bidirectionnelle — ci-dessus).
- **N:N** : **non géré comme mécanisme** par Fabrica. Se modélise comme une **entité de liaison
  ordinaire** (deux relations N:1), que l'**administrateur crée**. Bénéfices : l'entité de liaison
  peut porter ses **propres attributs** (type de relation, date…) et hérite des **colonnes système**
  (sys_id, dates, version, class — suivi/audit gratuits). Ne pas construire de cas spécial quand le
  cas général le couvre.

## Décision — intégrité référentielle et suppression

- **Invariant ferme : intégrité référentielle permanente.** Un objet **référencé ne peut pas
  disparaître** en laissant une référence pendante (FK en mode RESTRICT ; « invariants en base »,
  ADR-0004). L'intégrité est valide à tout moment.
- **Trois natures de données, trois régimes :**
  - **Données métier** : **pas de suppression physique**. Suppression **logique** via le booléen
    `actif` (ci-dessous). Jamais un objet référencé n'est effacé.
  - **Données techniques jetables** (logs, tables temporaires d'ingestion) : **purge physique**
    autorisée (non référencées par construction). Relève de la journalisation/ingestion.
  - **Données système** (métamodèle Fabrica + projet) : régime propre — cohabitation de deux versions
    puis ménage de la précédente (ADR-0014).
- **Archivage** : sujet réel, **pas d'ADR pour l'instant** (réservé).

## Décision — le booléen `actif`

- **`actif`** : booléen **obligatoire sur toute entité**, **vrai par défaut**. C'est la **suppression
  logique** (désactiver = `actif` faux) et le **marqueur de filtrage** fréquent (« objets actifs »).
- **Distinct du `state`** (cycle de vie, ADR-0019, capacité optionnelle) : `actif` répond à « cet
  objet compte-t-il encore ? » (existence logique) ; `state` répond à « où en est-il dans son
  processus métier ? ». Orthogonaux — un objet peut être `actif=vrai, state=clos`, ou `actif=faux`
  quel que soit son état.
- **Pas une colonne système** (il est *modifiable par l'utilisateur* — désactiver est une action
  métier), contrairement à celles de l'ADR-0020 : c'est un **attribut standard universel** (à
  rattacher au mécanisme des capacités d'entité — capacité toujours active).

## Hors périmètre (renvois)

- **Noms par défaut du côté N** : Fabrica peut dériver un libellé inverse par défaut (pluriel de
  l'entité source) ou exiger sa saisie — détail de l'ADR IHM/vues.
- **Relations N:N avec attributs** : couvertes par l'entité de liaison (ci-dessus), pas de mécanisme
  dédié.
- **Affectation / responsabilité**, **liens inter-entités** : usages de relations, ADR distincts.
- **Archivage** : réservé.

## Alternatives écartées

- **Traiter les listes de valeurs comme des relations** : perdraient la lisibilité par code (leur
  intérêt). Distinctes — critère volatilité/acteur ci-dessus.
- **N:N comme mécanisme du cœur** : complexité inutile ; l'entité de liaison ordinaire le couvre et
  offre attributs + colonnes système gratuitement.
- **Suppression physique d'objets métier** : casserait l'intégrité référentielle. Remplacée par
  `actif` (suppression logique).
- **Fusionner `actif` et `state`** : coupleraient deux concepts orthogonaux. Séparés.
