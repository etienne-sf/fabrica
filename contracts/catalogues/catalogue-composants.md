# Catalogue — Composants de rendu IHM

> Registre des composants de rendu. Mécanisme et définitions : **ADR product-0030**. Catalogue
> **fermé**, enrichi par Fabrica. Chaque composant respecte un **contrat d'interface** (reçoit
> valeur + état d'effets + locale ; rend ; déclare les types d'attributs qu'il gère) — ce qui
> prépare les **plugins projet** (réservés). Un attribut désigne son composant (défaut par type ;
> choix nécessaire pour les variantes).

| Type d'attribut | Composant(s) — édition | Cellule liste |
|---|---|---|
| chaîne mono-ligne | champ texte | texte tronqué |
| chaîne multi-ligne brute | zone de texte | texte tronqué |
| chaîne multi-ligne riche | éditeur riche | rendu tronqué |
| nombre | champ numérique (localisé) | nombre formaté, aligné droite |
| booléen | case à cocher / interrupteur | oui-non / icône |
| date / heure | sélecteur de date (localisé) | date formatée (localisée) |
| liste de valeurs | liste déroulante (options actives, labels traduits, filtrées si conditionnée) | label traduit |
| référence (objet) | autocomplétion display value + ouverture liste cible | display value |

> **Réservés (post-MVP)** : plugins de composants projet ; valeurs récentes d'un champ référence ;
> perso des colonnes de liste.
