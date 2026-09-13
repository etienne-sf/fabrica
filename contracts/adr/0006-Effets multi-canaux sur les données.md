# ADR-0006 — Effets multi-canaux sur les données : catalogue et régimes d'application

**Statut :** Proposé — 2026-07-11
**Portée :** décision du **cœur** (Fabrica), générique (Principe IV).
**Remplace** la version initiale de l'ADR-0006 (« effets d'IHM déclaratifs »), limitée au seul
canal IHM. À éclater en `contracts/adr/0006-effets-multicanaux.md`.

---

## Contexte

Un comportement conditionnel (interdire une modification, rendre obligatoire, masquer…) ne doit
pas être exprimé en termes d'**interface** : l'IHM n'est qu'un canal parmi d'autres (API,
ingestion…). La même intention métier doit valoir sur **tous les canaux**.

Les effets sont **déclenchés** par des règles (condition déclarative ou script) ; l'évaluation de
ces règles (itération jusqu'à point fixe, convergence, optimisation) relève du **moteur de règles**
(ADR distinct). Le présent ADR traite le *quoi* (les effets, leurs régimes d'application) ; le
moteur traite la *mécanique*.

## Principe du socle : le métamodèle est le plancher

Le **métamodèle** définit le socle de contraintes (types, obligation de base, contraintes
structurelles). **Aucun effet ne peut le contredire — un effet ne peut que restreindre davantage,
jamais desserrer.** Il peut rendre obligatoire un attribut optionnel au métamodèle ; il **ne peut
pas** rendre optionnel un attribut obligatoire, changer un type, ni lever une contrainte. Même
invariant que l'autorisation : **les couches successives resserrent, jamais n'ouvrent.** Fabrica
**vérifie au plus tôt** (validation du métamodèle) qu'aucun effet ne contrevient au socle.

## Nommage : par intention, jamais par apparence

Un effet se nomme par son **intention** (côté donnée si ancrage donnée, sinon intention de
présentation), **jamais par sa déclinaison visuelle**. « Griser » n'est **pas** un effet : c'est la
déclinaison *formulaire* de `interdire_modification`.

## Deux régimes d'application (et non « une colonne par canal »)

L'analyse par « colonne par canal » est écartée. Il n'existe que **deux régimes** :

- **Régime vérification (canal donnée) — le régime de base.** Toute **tentative de modification**,
  sur **n'importe quel canal** (API, ingestion, formulaire, liste en modification), déclenche
  l'évaluation des règles sur l'état résultant (données existantes + modification demandée). Le
  **verdict** accepte ou refuse ; un refus est accompagné d'un **message**. Aucun effet *visuel*
  n'est projeté — on vérifie, on tranche.
- **Régime projection (formulaire) — un supplément de confort.** Le **formulaire**, parce qu'il
  présente **un objet unique**, peut en plus évaluer les règles **à l'affichage** et **projeter**
  l'état des champs (grisé, caché…). C'est le seul canal qui projette, car il est le seul où
  l'évaluation a un coût borné (un objet) et un sens visuel (des champs à décliner).

**Critère de projection : un canal ne projette les effets que s'il présente un objet unique.** Dès
qu'un canal présente **plusieurs** objets (liste) ou **aucun** (API), il ne projette pas — il
**vérifie à la modification**.

### Le mode liste relève du régime vérification

Évaluer toutes les règles sur **chaque ligne** d'une grille serait prohibitif (temps *et* énergie).
Donc en liste : **aucun effet visible** (pas de cellule grisée ni masquée) ; les données sont
affichées brutes. Les règles ne s'évaluent **qu'au moment où l'utilisateur tente de modifier** une
ou plusieurs cellules ; si la modification est refusée, un **message** le lui indique. La liste se
comporte alors exactement comme le canal donnée.

## Matrice : donnée (autorité) et formulaire (projection)

| Effet | Donnée / vérification (autorité) | Formulaire (projection) |
|---|---|---|
| `interdire_modification` | Modification **refusée** (message). **Inconditionnelle** si déclenchée par règle d'intégrité : pas d'exception administrateur | Champ **grisé** (visible, non éditable) |
| `rendre_obligatoire` | Valeur **requise** (rejet si vide, message) | Champ marqué obligatoire ; vide refusé |
| `vider` | Valeur **effacée** (modification réelle de la donnée) | Champ vidé |
| `masquer` | **Aucun** (donnée **transmise**, non filtrée) | Champ **caché** |

Le **régime vérification** (colonne de gauche) est l'**autorité** ; le **formulaire** en est la
**projection de confort** (cohérent « serveur autoritaire, client projeté »). La liste et tous les
canaux non-objet-unique appliquent la colonne de gauche.

Notes :
- **`interdire_modification` vs `masquer`** — deux intentions : `interdire_modification` d'une valeur
  **pertinente mais figée** (date planifiée après validation → grisée en formulaire) ; `masquer` une
  valeur **sans sens dans ce contexte** (date planifiée pour une tâche non planifiée → cachée).
- **`masquer` en lecture** : donnée **transmise** (non filtrée). Masquage **purement présentationnel**.
  **Ne pas confondre avec l'autorisation** : cacher un champ en formulaire n'empêche pas l'API de
  l'exposer. Cacher *réellement* une donnée = **autorisation** (ADR-0012), pas cet effet.
- **`vider`** efface réellement la donnée.

## Intégrité vs autorisation

Un `interdire_modification` déclenché par une **règle d'intégrité** est une **contrainte de
cohérence, pas une autorisation** : il vaut pour **tous, administrateur inclus** (il protège la
cohérence des données, pas un privilège). À distinguer de l'autorisation (ADR-0012), qui dépend de
*qui* et que l'admin surpasse. Frontière à rappeler dans un ADR « règles d'intégrité ».

## Déclenchement et consommation de l'état final

- Déclencheur d'un effet : **condition déclarative** (choix dans une liste) ou **script** (une des
  entrées de la liste). Le *comment* du script → **ADR points d'entrée / scripts**.
- **Déclenchement fondamental = toute tentative de modification de donnée**, quel que soit le canal :
  elle déclenche l'évaluation et le verdict d'acceptation/refus.
- Les effets **consomment l'état final** produit par le **moteur de règles** : ils s'appliquent
  depuis le résultat stabilisé, jamais depuis les états intermédiaires. **Conséquence induite :
  aucun clignotement** (le formulaire ne voit que l'état final ; il ne passe pas par des états
  transitoires). Propriété induite par « le moteur produit, les effets consomment » — à préserver
  (ne jamais livrer d'états intermédiaires aux effets).

## Extensibilité

Catalogue **fermé, extensible par Fabrica**. Critère d'admission : le comportement de l'effet est
défini **pour les deux régimes** (vérification + projection formulaire). Démarrage **minimal**
(quatre effets), extension sur besoin réel.

## Alternatives écartées

- **Effets exprimés en termes d'IHM** (« griser », « cacher ») comme effets de premier rang :
  mono-canal. Remplacés par des intentions neutres.
- **Une colonne/régime par canal** : faux. Deux régimes seulement (vérification / projection),
  départagés par « objet unique ou non ». La liste est en régime vérification.
- **Projeter les effets en mode liste** : coût prohibitif (règles × lignes). Écarté : pas d'effet
  visible en liste, évaluation à la modification.
- **`rendre_optionnel`** : desserrerait le socle et n'a pas de sens sous « toutes les règles à
  chaque modification ». Retiré.
- **Filtrer la donnée d'un champ masqué en lecture** : confondrait masquage et confidentialité.
  Donnée transmise ; la confidentialité passe par l'autorisation.
- **Appliquer les effets au fil de l'évaluation** : clignotement + calcul non terminé. Remplacé par
  la consommation de l'état final.

## Renvois

- **Moteur de règles** (ADR à venir) : évaluation, point fixe, état final, non-convergence,
  optimisation. Le présent ADR *consomme* son résultat.
- **Points d'entrée / scripts** (ADR à venir) : déclenchement par script.
- **Traduction / i18n** (ADR à venir) : messages = clés + placeholders, jamais de texte en dur.
- **Sources de données / ingestion** (ADR à venir) : canal en régime vérification.
- **Règles d'intégrité** (ADR à venir) : frontière intégrité/autorisation.