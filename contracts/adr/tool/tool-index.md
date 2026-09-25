# Index des ADR — Fabrica / **Outillage** (tool)

> Famille **Outillage** : décisions sur *comment on fabrique, package, déploie et exploite* un
> projet Fabrica. Distincte de la famille **Produit** (`../product/`), qui porte sur le comportement
> de Fabrica et de l'application générée. Numérotation propre à la famille (`tool-NNNN`). Les numéros
> sont des identifiants, pas un ordre de lecture. À vérifier contre le dépôt git.

## Cycle de vie & métamodèle
- **tool-0001** (P) Cycle de vie d'un projet, **auto-application** (Fabrica édite son métamodèle),
  **bootstrap** (méta-métamodèle), **source de vérité git**, **format JSON canonique** par entité,
  frontière métamodèle / seed / données de production.
- **tool-0002** (P) **Revue de montée de version** : impacts bloquant/info, auto-détection, blocage
  du packaging, tableau de bord (MVP). ↔ product-0030 (rendu de l'écran).

## Numéros
- Prochain libre : **tool-0003**.

## Dettes — ADR outillage à écrire
- **Développement concurrent** (résolution de conflits) & **affichage des écarts** entre versions.
- **Droits dans l'outil de développement** (2ᵉ système d'autorisation, distinct du RUN).
- **Définition de tests par le projet**.
- **Haute disponibilité** (architecture générée).
- **Exploitation** (sauvegarde, clonage d'environnement, monitoring…).
- **Outillage par phase** (générateur, packager, déployeur) — détail au besoin.
