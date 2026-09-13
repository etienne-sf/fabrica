# ADR-0015 — Moteur de règles : composant générique encapsulé

**Statut :** Proposé — 2026-07-11
**Portée :** décision du **cœur** (Fabrica), générique (Principe IV). Composant utilisé par le
moteur des effets (ADR-0016). À éclater en `contracts/adr/0015-moteur-de-regles.md`.

---

## Contexte

Le moteur des effets (ADR-0016) a besoin d'évaluer un ensemble de règles sur un état donné. Cette
évaluation est un problème **standard** (pattern *rules engine* : des règles condition → conclusion,
un moteur d'inférence, un appelant qui agit). Il n'y a aucune raison d'en réécrire un.

## Décision

**Fabrica s'appuie sur un moteur de règles générique, agnostique — de préférence une bibliothèque
existante et éprouvée — plutôt que d'en développer un.**

Le moteur de règles est :
- **agnostique** : il ne connaît ni la notion d'effet, ni les données de Fabrica, ni l'itération ni
  le point fixe (portés par le moteur des effets). Il pourrait servir à autre chose ;
- **sans état et sans effet** : il fait **une passe** — il n'écrit rien, ne boucle pas ;
- **encapsulé et remplaçable** : Fabrica ne dépend de lui que par un **contrat mince** ; on peut en
  changer sans toucher au moteur des effets. Même principe d'isolation que Grafast (ADR-0011 ; ADR-0007
  appliqué à un composant tiers).

## Contrat attendu (le contrat mince)

- **Entrée** : un **état** (les données pertinentes) + un **ensemble de règles** (condition →
  conclusion).
- **Sortie** : les **conclusions** d'une passe — quelles règles se sont déclenchées et ce qu'elles
  concluent (effets à poser, modifications de données à faire). Le moteur des effets interprète ces
  conclusions (les modifications relancent la boucle, les effets sont accumulés — ADR-0016).
- **Aucun effet de bord** : le moteur ne modifie pas l'état, ne persiste rien, ne rappelle personne.

## Modèle de « règle »

Une règle est **condition → conclusion(s)**. La condition est **déclarative** (évaluable par le
moteur) ou **scriptée** (déléguée — le moteur reçoit le verdict d'un script exécuté par Fabrica, cf.
ADR points d'entrée). Les conclusions sont neutres pour le moteur : il les rend, il ne sait pas ce
qu'elles *font* (c'est le moteur des effets qui sait qu'une conclusion est un `vider` ou un
`masquer`). Le modèle uniforme de règle est le **contrat d'entrée** du moteur, ce qui le rend
générique.

## Choix de la bibliothèque (différé)

Le choix du moteur de règles concret (bibliothèque, version) est **différé à l'implémentation** :
c'est un composant remplaçable, non structurant tant que le contrat mince ci-dessus est respecté.
Critères à ce moment : agnosticité, licence open source (cohérent exigence Fabrica), maturité,
capacité à évaluer une passe sans effet de bord, intégration avec l'écosystème retenu
(TypeScript/JavaScript). À vérifier alors, pas de mémoire.

## Alternatives écartées

- **Développer un moteur de règles maison** : réécrit un composant standard et mûr ; effort
  disproportionné. Écarté au profit de l'encapsulation d'un existant.
- **Mettre le point fixe / l'état dans le moteur de règles** : le rendrait spécifique à Fabrica et
  non remplaçable. Le point fixe est dans le moteur des effets (ADR-0016).

## Conséquences

- Le moteur de règles est un **détail d'implémentation encapsulé** : son remplacement n'impacte ni le
  moteur des effets, ni les projets.
- La détection de non-convergence et le traitement des contradictions **ne sont pas** de son ressort
  (il ignore la boucle) — ils sont dans le moteur des effets (ADR-0016).