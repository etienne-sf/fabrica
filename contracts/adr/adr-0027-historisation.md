# ADR-0027 — Historisation des changements (capacité `historiser`)

**Statut :** Proposé — 2026-07-11
**Portée :** décision du **cœur** (Fabrica), générique (Principe IV). Capacité d'entité (mécanisme :
ADR-0025). Distincte de l'**audit de traçabilité** (« T » de DICT, ADR distinct) et de la
**journalisation technique** (ADR distinct). À éclater en `contracts/adr/0027-historisation.md`.

---

## Contexte

L'historisation garde l'**histoire des changements métier** d'un objet — « qui a changé quoi,
quand » — pour les **utilisateurs**. À ne pas confondre avec l'**audit** (sélectif, sécurité,
pour la conformité — recoupement possible sur la capture, finalités distinctes) ni les **logs
techniques** (exploitants). La capacité était nommée `auditer` ; **renommée `historiser`** pour
lever l'ambiguïté.

## Décision — capacité `historiser`, par attribut

- `historiser` est une **capacité d'entité optionnelle** (ADR-0025), activée par entité.
- **Choix par attribut** : on désigne **quels attributs** sont historisés (le défaut de l'entité
  peut être surchargé par attribut). Limite le coût (cf. performance).
- **Besoin retenu au démarrage : « qui a changé quoi »** (historique des changements). La
  **reconstitution d'état à une date passée** (versioning temporel) est **réservée** comme capacité
  future distincte.

## Décision — structure : une entrée par modification, une ligne par attribut

- Une **modification** (un acte de sauvegarde, une transaction) = **une entrée** d'historisation
  (utilisateur, date, objet).
- Chaque attribut historisé modifié dans cet acte = **une ligne** (attribut, valeur avant, valeur
  après). Un acte qui change trois attributs → une entrée, trois lignes.
- **Ordre d'affichage des lignes = ordre des attributs dans la définition de l'entité** (ce qui
  implique que l'**ordre des attributs est une propriété du métamodèle** — consommée aussi par
  formulaires et listes).
- **Stockage dans une table d'historique séparée** — **jamais** dans la table active (empiler des
  versions dans la table active casserait les clés primaires et les FK ; l'intégrité référentielle
  est préservée en gardant l'historique à côté).

## Décision — valeurs stockées brutes, rendues selon le lecteur

- On stocke les **valeurs brutes** (nombre, date, code de liste, `sys_id` pour une référence).
- L'**affichage** est une **projection contextuelle**, rendue **selon l'utilisateur qui lit**
  (jamais selon celui qui a modifié) : formats de nombre/date de l'utilisateur, label de liste
  traduit, référence résolue en **valeur d'affichage** (display value) actuelle de l'objet cible.
  Mécanisme de formatage : ADR-0017 (formatage localisé des valeurs).
- **Fidélité au fait, pas au libellé** : un objet référencé renommé s'affiche sous son nom actuel —
  ce qui compte est *quel objet* a été associé (le `sys_id`), pas le nom d'alors.
- Cas **ajout / suppression / remplacement** (valeur vide de/vers) gérés distinctement (pas de
  multivalué à traiter — interdit par la constitution).
- Le message est **traduit** (i18n : clé + placeholders, ADR-0017).

## Décision — effets induits : utilisateur dans la transaction, système hors

- Les modifications **induites par les effets dans la transaction** (cascades `apresModif`,
  `vider` en chaîne… jusqu'au point fixe avant commit — ADR-0016) sont historisées **au nom de
  l'utilisateur** : c'est son acte atomique qui les a déclenchées.
- Les modifications **post-transaction** (`apresTransaction`, asynchrone, après commit) sont
  historisées **au nom du système** (l'identité de session n'est plus active). Aligné sur le QUI
  de l'ADR-0012 et `sys_updated_by` (ADR-0020).

## Décision — capture par trigger, vérifiable

- **Capture par trigger PostgreSQL** (after) sur les attributs marqués : au plus près de la donnée,
  **tous chemins** (un logging applicatif raterait les changements faits en base, en batch, en
  script). Cohérent « invariants en base » (ADR-0004).
- **Anti-trou (crainte légitime : trigger désactivé et oublié) :**
  1. le **compte applicatif** de Fabrica **n'a pas le droit de désactiver** les triggers → aucune
     opération du flux applicatif ne peut les couper ;
  2. **vérification périodique** (au démarrage et régulièrement) que tous les triggers d'historisation
     attendus sont **actifs** ; sinon **alerte** (le trou n'est jamais silencieux — principe DICT
     « capture vérifiable », constitution).
- **Niveau supérieur (réservé, hors Fabrica)** : la capture par **CDC** (lecture du WAL, non
  désactivable) se met en place **à l'instanciation, au niveau infrastructure**, si le niveau de
  traçabilité l'exige. Le CDC peut assurer la traçabilité **générale**, pas seulement surveiller les
  triggers. Fabrica ne le porte pas.

## Décision — performance

- **Capture** : limitée aux **attributs marqués** (pas toute l'entité) — le coût principal.
- **Consultation** : l'historique et la **résolution des display values** se font en **une seule
  requête** (anti-N+1 via Grafast, ADR-0011), chargée **à la demande / asynchrone** (l'historique
  n'est pas dans le chemin critique).

## Hors périmètre (renvois / dettes)

- **Valeur d'affichage (display value)** : **capacité obligatoire** de toute entité (représentation
  d'affichage d'un objet). Consommée ici et par les champs Référence, listes, recherche. → catalogue
  capacités ; réalisation possible via un **attribut calculé** (« prénom nom »).
- **Attributs calculés / dérivés** (recalculés à la modification) : dette (le display value en est le
  premier cas).
- **Ordre des attributs** : propriété de la définition d'entité (à acter dans l'ADR entités/métamodèle).
- **Audit de traçabilité** (« T » de DICT) : ADR distinct — automatique, par entité et par attribut
  (le niveau de l'entité donne le défaut de l'attribut, surchargeable ; par ligne réservé).
- **Journalisation technique** : ADR distinct.
- **Reconstitution d'état temporel** : capacité future.
- **Externalisation de la capture** : via les mécanismes post-modification / CDC (infra).

## Alternatives écartées

- **Historique dans la table active** (versions empilées) : casse PK/FK. Table séparée.
- **Figer le libellé au moment du changement** : fige une traduction, alourdit, et affiche un nom
  mort. Remplacé par « stocker le `sys_id`, résoudre le display value à la lecture ».
- **Logging applicatif** (au lieu de trigger) : rate les changements hors application. Trigger base.
- **Tout historiser** (toute l'entité) : coût. Par attribut marqué.
