# ADR-0021 — Re-classification d'un objet (changement de sys_class)

**Statut :** Différé (non traité en v1) — 2026-07-11
**Portée :** décision du **cœur** (Fabrica). Identifie et borne le sujet ; ne le tranche pas.
À éclater en `contracts/adr/0021-reclassification.md`.

---

## Contexte et décision

Changer la classe réelle d'un objet (`sys_class`, ADR-0020) — ex. reformater un serveur Linux en
Windows dans une CMDB — implique, dans le modèle d'héritage « table par classe, une ligne par
niveau » (ADR-0005), de **déplacer la ligne** : retirer la ligne du niveau quitté, créer celle du
niveau rejoint, en conservant `sys_id` et la ligne racine.

**Décision : non traité en v1.** Le sujet est **identifié et évacué**, pas résolu.

## Ce que le sujet tirera (quand il sera traité)

- **Transitions de classe autorisées** : peut-on passer de n'importe quelle classe à n'importe
  quelle autre, ou seulement entre classes compatibles (sœurs sous un même parent) ? Ressemble à un
  cycle de vie appliqué à la classe (cf. ADR-0019).
- **Migration des attributs** : les attributs spécifiques au niveau quitté disparaissent ; ceux du
  niveau rejoint doivent être renseignés (potentiellement obligatoires). Ce n'est pas un simple
  `update`.
- **Atomicité multi-tables** : suppression d'un niveau + création d'un autre, en une transaction
  (orchestration d'écriture, ADR-0005/0016).
- **Autorisation / intégrité** : qui peut re-classifier, et sous quelles règles.

Non prérequis v1.
