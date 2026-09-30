# Catalogue — Composants de rendu IHM

> Registre des composants de rendu. Mécanisme et définitions : **ADR product-0030**. Catalogue
> **fermé**, enrichi par Fabrica. Chaque composant respecte un **contrat d'interface** (reçoit
> valeur + état d'effets + locale ; rend ; déclare les types d'attributs qu'il gère) — ce qui
> prépare les **plugins projet** (réservés). Le composant se déduit du type et du
> sous-type de l'attribut (ADR-0037) ; la surcharge dans une vue est post-MVP.

| Type / sous-type | Composant — édition | Cellule liste |
|---|---|---|
| `text/line` | champ texte | texte tronqué |
| `text/multiline` | zone de texte | texte tronqué |
| `text/rich` | éditeur riche | rendu tronqué |
| `number/integer` | champ numérique (localisé) | nombre formaté, aligné à droite |
| `number/decimal` | champ numérique (localisé, échelle) | nombre formaté, aligné à droite |
| `boolean` | case à cocher | oui-non / icône |
| `temporal/date` | sélecteur de date (localisé) | date formatée |
| `temporal/time` | sélecteur d'heure (localisé) | heure formatée |
| `temporal/datetime` | sélecteur date-heure (fuseau de l'utilisateur) | date-heure formatée |
| `valuelist` | liste déroulante (options actives, labels traduits, filtrées si conditionnée) | label traduit |
| `reference` | autocomplétion sur la valeur d'affichage + ouverture de la liste cible | valeur d'affichage (objet restreint si non accessible) |

> **Réservés (post-MVP)** : plugins de composants projet ; valeurs récentes d'un champ référence ;
> perso des colonnes de liste.
