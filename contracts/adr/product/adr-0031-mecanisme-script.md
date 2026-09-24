# ADR-0031 — Mécanisme de script : contexte, sandbox, contrat (noyau)

**Statut :** Proposé — 2026-07-11
**Famille :** **Produit** (`product/`). **Noyau** du mécanisme d'exécution des scripts projet — sujet
fondateur destiné à **s'enrichir progressivement** (ce noyau MVP couvre le strict nécessaire).
Générique (Principe IV). **Versant tool :** mise à disposition / compilation du code projet (dette).

---

## Contexte et changement de périmètre

On avait posé « pas de script au MVP ». Le **display value** (« prénom nom ») prouve que le MVP a
besoin de *calcul* dès le premier besoin réel. Or construire un **catalogue d'opérations maison**
(concaténer, formater, arithmétique…) reviendrait à réimplémenter TypeScript en moins bien — du
travail **jetable**. On **remet donc les scripts à l'endroit** : le **noyau** du mécanisme de script
entre au MVP. On ne ramène **que le noyau** (contexte + sandbox + contrat + retour), pas tout le
confluent (accès base, intégrations, conformité avancée restent hors MVP).

## Décision — exécution en sandbox partant d'un contexte vide

- Fabrica appelle un **script TypeScript** (compilé en JavaScript), exécuté dans un **sandbox** qui
  part d'un **contexte vide** : **aucun module n'est importable** — les imports externes (`fs`,
  réseau, packages) échouent **par construction** (il n'y a pas de système de modules à résoudre, pas
  parce qu'on les « interdit »).
- **Fonctions pures du langage disponibles** (manipulation de chaînes, etc.) — c'est ce qui évite de
  réimplémenter un langage d'expressions. **Aucun accès ambiant** : ni variables d'environnement, ni
  réseau, ni base. La sécurité tient à **deux** garanties conjointes : le sandbox (rien par défaut)
  **et** la discipline de Fabrica (elle n'injecte que le nécessaire, jamais d'accès dangereux) — le
  sandbox seul ne suffit pas si Fabrica injecte mal.

## Décision — le contexte injecté (mécanisme partagé)

- Fabrica injecte dans le sandbox un **contexte** contrôlé. Pour ce noyau MVP, la seule donnée de
  contexte est l'**objet courant** (ses attributs).
- **C'est *le* mécanisme de contexte, pas un mécanisme spécifique au display value** : les crochets
  `beforeUpdate` / `afterUpdate` (et les futurs points d'entrée) réutiliseront le **même** mécanisme,
  avec un contexte adapté. Il est donc conçu comme fondation partagée.

## Décision — contrat d'exécution

- **Entrée** : le contexte (objet courant).
- **Sortie** : une valeur — **texte uniquement au MVP**.
- **Valeurs nulles → chaîne vide** (rendu prévisible, sans nettoyage — ne pas sur-concevoir).
- **Script autonome, sans import** : c'est le besoin MVP **et le cas général** (un script Fabrica est
  une **fonction** branchée sur un point d'entrée — reçoit un contexte, rend un résultat — pas un
  programme ; il n'a presque jamais besoin d'importer). L'**organisation multi-fichiers** du code
  projet est l'exception, **réservée**.

## Hors périmètre (renvois / dettes)

- **Mise à disposition / compilation du code projet** (comment le dev livre sa librairie TypeScript,
  compilée, embarquée dans l'image ; conformité ; complétude — tout point d'entrée déclaré est
  implémenté) : **versant tool**, dette — lien bidirectionnel.
- **Accès base depuis un script**, **intégrations / services externes** (avec accès au monde,
  variables d'environnement…), **autres types de contexte**, **messages traduits** (clés + params),
  **retour non-texte**, **organisation multi-fichiers** : le reste du **confluent**, post-MVP.
- **Points d'entrée du cycle de données** (`beforeUpdate`/`afterUpdate`…) : ADR à venir, réutilisent
  ce noyau.

## Alternatives écartées

- **Catalogue d'opérations maison** (mini-langage d'expressions) : réimplémente TypeScript, travail
  jetable, puits sans fond. Remplacé par le script pur.
- **Script non borné** (accès environnement/réseau/base, tous modules) : rouvre les dangers
  (contournement d'autorisation, fuite de secrets, risque conteneur). Remplacé par le script **pur en
  sandbox** (fonctions pures oui, accès ambiant non).
- **Système d'imports/modules pour le projet** : inutile — le script autonome est le cas général.

## Premier client

L'**attribut calculé / display value** (ADR-0032) est le premier usage de ce noyau.
