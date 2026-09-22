# ADR-0022 — Collaboration temps réel

**Statut :** Différé (non traité en v1) — 2026-07-11
**Portée :** décision du **cœur** (Fabrica). Identifie et borne le sujet ; ne le tranche pas.
À éclater en `contracts/adr/0022-collaboration-temps-reel.md`.

---

## Contexte et décision

Quand deux utilisateurs affichent le même objet et que l'un le modifie, l'autre devrait voir la
nouvelle version **en direct** (mécanisme apprécié de ServiceNow). Techniquement : subscription
GraphQL, ou bus de messages, via **websockets** (avec repli/reconnexion si la connexion tombe).

**Décision : non traité en v1.** Le sujet est **identifié et évacué**, pas résolu.

## Ce que le sujet tirera (quand il sera traité)

- Choix du transport (subscriptions GraphQL vs bus) et sa cohérence avec le moteur GraphQL (ADR-0011).
- Robustesse : repli, reconnexion, reprise après coupure.
- **Interaction avec le verrouillage optimiste** (ADR-0020) : voir la nouvelle version en direct
  **réduit les collisions** au lieu de seulement les refuser (l'autre utilisateur a déjà la version à
  jour au moment de sauver). Le temps réel *adoucit* le lock optimiste.

Non prérequis v1.
