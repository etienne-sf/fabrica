# ADR-0030 — Rendu de l'IHM : interprète dynamique et catalogue de composants

**Statut :** Proposé — 2026-07-11
**Famille :** **Produit** (`product/`). Réalise le **contrat d'IHM** (ADR-0003) : celui-ci dit *quels
attributs, quel ordre* ; le présent ADR dit *comment ça s'affiche*. Générique (Principe IV).
**Versant tool :** l'**éditeur de vues** (composer formulaires/listes) — ADR tool à venir (dette).

---

## Contexte

L'ADR-0003 pose le contrat d'IHM piloté par le métamodèle, sans le rendu concret. Cet ADR le traite,
pour le MVP : formulaires et listes affichés à l'utilisateur (pas d'intégration SI, pas de tableaux
de bord — voir hors périmètre).

## Décision — rendu dynamique (interprète), pas génération statique

L'IHM est produite par un **interprète générique du métamodèle**, à l'exécution — **pas** du code
front généré par projet et buildé. Raison décisive : **testabilité.** Un interprète est **testé une
fois** et vaut pour **tous** les métamodèles ; du code généré par projet ne serait jamais testé en
tant que tel (on ne pourrait tester que le générateur, pas ses sorties infinies). C'est le Principe II
appliqué au front : tester un interprète fini + valider des données, plutôt que tester du code
généré à l'infini. Cohérent aussi avec le métamodèle réflexif (ADR-0019) et « minimiser le bespoke ».

## Décision — technologie

- **TypeScript** (typé, compilé en JavaScript pour le navigateur ; unité avec le code projet,
  ADR-0017). Alternatives (JS nu, WASM, Elm/ReScript/Dart) écartées : typage perdu ou écosystème UI
  immature/de niche.
- **React** retenu (écosystème de composants — data grids, form renderers — le plus riche, ce qui
  sert le **build-vs-buy**).
- **Build vs buy** : les composants (grille, champs) s'appuient sur des **bibliothèques React
  éprouvées** ; l'interprète Fabrica **orchestre**, il ne réimplémente pas une data grid.

## Décision — catalogue de composants (fermé, à contrat d'interface)

- **Catalogue fermé** de composants de rendu, **enrichi par Fabrica** (constitution § Catalogues) →
  `contracts/catalogues/`.
- **Chaque composant respecte un contrat d'interface défini par Fabrica** : il reçoit la **valeur**,
  l'**état calculé par les effets**, la **locale** (ADR-0017) ; il **rend** ; il **déclare** quels
  types d'attributs il sait rendre. Ce contrat est la **frontière calée** qui rendra l'ouverture
  future aux **plugins de composants projet** simple (un plugin = une entrée respectant le contrat —
  post-MVP, priorité faible ; un plugin de composant sera un cas des points d'entrée).
- **Un attribut désigne son composant** parmi le catalogue (**défaut sensé par type** si non
  précisé ; le choix est **nécessaire dès le MVP** pour départager les variantes d'un même type —
  ex. chaîne mono-ligne / multi-ligne / riche). Les **plugins externes** (hors catalogue) sont
  **réservés**.

## Décision — mapping type d'attribut → composant

| Type d'attribut | Composant formulaire (édition) | Cellule liste (affichage) |
|---|---|---|
| chaîne **mono-ligne** | champ texte | texte (tronqué) |
| chaîne **multi-ligne brute** | zone de texte | texte (tronqué) |
| chaîne **multi-ligne riche** | éditeur avec mise en forme | texte/rendu (tronqué) |
| **nombre** | champ numérique (format localisé) | nombre formaté (localisé), aligné à droite |
| **booléen** | case à cocher / interrupteur | oui-non (ou icône) |
| **date / heure** | sélecteur de date (localisé) | date formatée (localisée) |
| **liste de valeurs** | **liste déroulante** — options = valeurs **actives**, labels traduits, **filtrées** si la liste est conditionnée (ADR-0018, via le moteur des effets) | label traduit de la valeur |
| **référence (objet)** | **autocomplétion sur la display value** (on tape, on cherche, on stocke le `sys_id`) **+ ouverture de la liste de l'entité cible** pour choisir | **display value** de l'objet cible |

- Les **trois variantes de chaîne** sont **tronquées en liste** (une liste ne montre jamais un pavé).
- **Référence vs liste de valeurs** : recherche/autocomplétion (objet volatil, par `sys_id`) vs
  déroulante (domaine borné, par code) — la traduction visuelle de la distinction de l'ADR-0018/0023.
- **Champ référence, réservé (post-MVP)** : **valeurs récemment utilisées** par l'utilisateur (tire un
  mécanisme d'historique/préférences par utilisateur — donnée de production) ; **personnalisation des
  colonnes** de la liste cible.
- Formatage localisé : ADR-0017 (nombres/dates selon les préférences de l'utilisateur qui affiche).

## Décision — états des composants pilotés par les effets

Un composant ne fait pas qu'afficher une valeur : il l'affiche **dans l'état calculé par le moteur des
effets** (ADR-0016) — éditable / **grisé** (`interdire_modification`) / **caché** (`masquer`) /
**obligatoire** (`rendre_obligatoire`) / **vidé** (`vider`). Le rendu est la **projection**
formulaire/liste de l'état final (régimes de l'ADR-0006 : le formulaire projette, la liste applique le
régime donnée à la modification). Anti-clignotement hérité (ADR-0016 : seul l'état final est appliqué).

## Décision — formulaires et listes (MVP)

- **Formulaire** : **un par entité** au MVP. En l'absence de définition, Fabrica affiche un
  **formulaire par défaut** — tous les attributs, dans l'**ordre déclaré** (ADR : l'ordre est une
  propriété du métamodèle). Ce défaut est aussi le **point de départ** proposé au développeur quand
  il crée ou réinitialise un formulaire → c'est un **gabarit fourni** (constitution).
- **Liste** : les colonnes affichées sont la **définition projet** (l'utilisateur les subit au MVP,
  dans l'ordre déclaré). La **personnalisation par utilisateur** (choix/ordre des colonnes) est une
  **surcharge réservée** (place calée, hors MVP) — l'ADR distingue *définition projet* (défaut) et
  *préférence utilisateur* (surcharge) pour ne pas coder « colonnes = projet » en dur.

## Hors périmètre (renvois / dettes)

- **Éditeur de vues** (composer formulaires/listes) : **versant tool**, ADR tool à venir — lien
  bidirectionnel avec le présent ADR.
- **Tableaux de bord / reporting** : MVP mais **différable** (on peut générer sans). *Mémo* : ils
  tirent le **seed** (livrer un dashboard à une montée de version = init de données) et les **dettes
  de migration** (un changement de modèle peut casser un report — filtre à faire évoluer). ADR
  reporting distinct.
- **Plugins de composants projet** : réservés (le contrat d'interface les prépare).
- **Perso des colonnes de liste**, **valeurs récentes d'un champ référence** : réservés (préférences
  utilisateur).
- **Intégration SI** (API exposée, import) : hors MVP.

## Alternatives écartées

- **Génération statique du front** (code par projet, buildé) : jamais testé en tant que tel ; rebuild
  à chaque changement de métamodèle. Remplacée par l'interprète dynamique (testé une fois).
- **Composants en dur (switch sur le type)** : fermerait l'ajout futur de plugins. Remplacé par un
  catalogue à contrat d'interface.
- **Déroulante pour une référence** : ingérable (charger des milliers d'objets). Autocomplétion.
- **Recherche pour une liste de valeurs** : surdimensionné (domaine borné). Déroulante.
