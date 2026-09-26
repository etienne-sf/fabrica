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
- **tool-0003** (P) **Mise à disposition du code, arborescence du dépôt, git, exécution locale** :
  script créé par Fabrica, surcharges d'entités Fabrica, validations MVP, deux modes de compilation.
  ↔ product-0031 (mécanisme de script).

## Numéros
- Prochain libre : **tool-0004**.

## Dettes — ADR outillage à écrire
- **Développement concurrent** (résolution de conflits) & **affichage des écarts** entre versions.
- **Droits dans l'outil de développement** (2ᵉ système d'autorisation, distinct du RUN).
- **Définition de tests par le projet**.
- **Haute disponibilité** (architecture générée).
- **Exploitation** (sauvegarde, clonage d'environnement, monitoring…).
- **Outillage par phase** (générateur, packager, déployeur) — détail au besoin.
- **Alternatives à Docker** pour postes sans droits d'administrateur (Podman rootless, env. distant, natif).
- **Pilote de fusion sémantique** ; **validateur étendu** (canonicité, orphelins) ; **crochets git**.
