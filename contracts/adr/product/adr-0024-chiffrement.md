# ADR-0024 — Chiffrement des données

**Statut :** Différé (non traité en v1) — 2026-07-11
**Portée :** décision du **cœur** (Fabrica). Réalise le « C » (Confidentialité) du cadre DICT de la
constitution. Identifie et borne le sujet ; ne le tranche pas. À éclater en
`contracts/adr/0024-chiffrement.md`.

---

## Contexte et décision

Le chiffrement contribue à la **Confidentialité** (cadre DICT, constitution). Le sujet tire de
nombreuses questions et n'est pas un prérequis v1.

**Décision : non traité en v1.** Sujet **identifié et évacué**, pas résolu.

## Ce que le sujet tirera (quand il sera traité)

- **Portée du chiffrement** : au repos (base, disques), en transit (déjà largement couvert par
  TLS), **par champ** (chiffrer seulement certains attributs sensibles).
- **Gestion des clés** : stockage, rotation, séparation des rôles — souvent le point le plus dur.
- **Interaction avec les mécanismes du cœur** : un champ chiffré est-il requêtable, filtrable,
  indexable ? (le chiffrement par champ casse le pushdown SQL, la RLS, la recherche — arbitrage
  confidentialité vs exploitabilité).
- **Mesurabilité** (exigence DICT) : constater/monitorer que les données d'un niveau de
  confidentialité donné sont effectivement chiffrées.
- **Configurable par criticité du projet** (comme la gouvernance d'extraction, ADR-0009) : un
  référentiel EA n'a pas les mêmes besoins qu'une CMDB sensible.

Non prérequis v1.
