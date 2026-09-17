# ADR-0017 — Internationalisation : clés, espace de noms, langues

**Statut :** Proposé — 2026-07-11
**Portée :** décision du **cœur** (Fabrica), générique (Principe IV). Transverse : concerne tout ce
qui est **affiché à l'utilisateur**. Définit le **contrat de message** que les points d'entrée
devront respecter (ADR scripts à venir). À éclater en `contracts/adr/0017-internationalisation.md`.

---

## Contexte

Tout texte présenté à l'utilisateur (libellés d'entités et d'attributs, valeurs de listes, textes
d'interface, messages) doit être **traduisible**. Un texte n'est jamais en dur : c'est une **clé**
résolue selon la langue. L'i18n est un domaine mûr ; Fabrica **s'appuie sur une bibliothèque i18n
standard** plutôt que de réimplémenter le rendu.

## Décision — délégation à une bibliothèque i18n

Fabrica **orchestre** (d'où vient la langue, comment les clés sont attachées au métamodèle et aux
scripts) ; la **bibliothèque i18n rend** (clé + paramètres + langue → texte final, avec placeholders
et pluralisation). Choix de la bibliothèque **différé à l'implémentation** (critères : open source,
maturité, placeholders nommés, pluralisation, écosystème TypeScript/JavaScript). Même principe
d'encapsulation que Grafast (ADR-0011) et le moteur de règles (ADR-0015).

## Décision — langue

- **Langue courante** = défaut de l'**instance**, surchargé par la **préférence utilisateur** si
  définie. Modifiable à tout moment via les préférences ; l'écran affiché **se rafraîchit** dans la
  nouvelle langue (même mécanisme de re-rendu que le déploiement à chaud, ADR-0014).
- La langue voyage dans le **contexte** de requête (comme l'identité, ADR-0010) : Fabrica en a besoin
  au moment de résoudre une clé.
- **Anglais obligatoire** à la création de toute clé, et **valeur par défaut**. Conséquence : toute
  clé a **toujours** une valeur anglaise → le **fallback** est en cascade (langue demandée, puis
  anglais garanti) ; **jamais** de clé brute affichée.

## Décision — espace de noms des clés (anti-collision)

Les clés sont organisées en **familles** (préfixe de famille, en anglais), qui isolent les natures
et **portent les droits de création** :

| Famille | Contenu | Création |
|---|---|---|
| `entity` | libellés d'entités et d'attributs (métamodèle) | **dérivée** du métamodèle ; création manuelle **interdite** |
| `list` | valeurs des listes de référence | **dérivée** du métamodèle ; création manuelle **interdite** |
| `system` | textes livrés par Fabrica (rôles standards, colonnes système, messages du cœur) | **réservée Fabrica** ; création manuelle **interdite** |
| `message` | messages des scripts et du cœur (avec placeholders) | libre (projet et cœur) |
| `ui` | textes d'interface hors métamodèle (boutons, menus, titres) | libre (projet) |

**Structure des clés** (le code technique est la **racine** de la clé, pas son contenu) :
- entité/attribut : `entity:<code_entité>.<code_attribut>.<texte>` — ex. `entity:u_task.due_date.label`.
- entité elle-même : `entity:<code_entité>.<texte>`.
- valeur de liste : `list:<code_entité>.<code_attribut>.<valeur>.<texte>` — la liste est **nommée par
  l'attribut qu'elle qualifie** (garantit l'unicité, évite d'inventer un nom de liste).

## Décision — préfixes de code (séparation projet / Fabrica dans le temps)

Pour empêcher toute collision — y compris **dans le temps** (une future version de Fabrica
introduisant une entité de même nom qu'une entité projet existante) :

- **Tout code d'entité porte un préfixe** ; l'espace **sans préfixe est interdit**.
- **Fabrica maintient la liste des préfixes qu'elle se réserve** (`sys_` au démarrage, et d'autres à
  venir pour de futurs modules distincts). Le projet ne peut utiliser **aucun** préfixe réservé.
- Le projet utilise **`u_`** (entités *custom*). Avantage assumé : `u_` **met en avant** ce qui est
  custom (propriété visible d'un coup d'œil), ce qui est utile dans un métamodèle mixte
  Fabrica+projet.
- Conséquence : Fabrica ne créera jamais `u_task` ni `task` (nu), le projet ne créera jamais
  `sys_task` — la collision temporelle est **impossible par construction**, pas seulement atténuée.

## Décision — textes gérés (démarrage minimal)

- **`label`** (obligatoire) : l'intitulé.
- **`description`** (optionnelle) : affichée en **popin au survol** (sur le champ en formulaire, sur
  le titre de colonne en liste).
- Ces textes sont **statiques** : pas de pluriel (idem valeurs de listes).
- **Pluralisation** : disponible (via la bibliothèque) pour la famille **`message`** uniquement (ex.
  « {n} tâche(s) »), sans objet pour les libellés statiques.
- Extensible : d'autres textes (libellé court…) s'ajouteront au besoin.

## Décision — placeholders

**Nommés** (`{seuil}`), pas positionnels (`{0}`). Plus lisibles pour le traducteur et robustes au
**réordonnancement** selon les langues (l'ordre des mots varie). Surcoût quasi nul (nativement gérés
par les bibliothèques i18n) ; il ne porte que sur le **contrat** (fournir `{ nom: valeur }`).

## Décision — pas de surcharge au démarrage

Le projet fournit les traductions de **ses** clés et consomme celles de Fabrica **telles quelles**.
Il ne peut **pas** redéfinir une traduction Fabrica (**surcharge** non construite).
- Conséquence assumée : dans une langue où Fabrica n'a pas traduit ses propres libellés (`system`,
  `entity`/`list` du cœur), ces textes retombent en **anglais**, et le projet ne peut pas y remédier.
- La **surcharge est réservée** comme point d'extension **additif** (ADR-0013) : l'ajouter plus tard
  **ne casse aucun contrat** (élargit ce que le projet peut faire). Son **déclencheur naturel** : un
  projet ayant besoin d'une langue que Fabrica ne fournit pas pour ses libellés.

## Contrat de message (pour les points d'entrée / scripts)

Un script (ou le cœur) ne rend **jamais de texte en dur**. Il rend, en plus de son résultat
fonctionnel (ok / avertissement / ko — ADR-0006), une liste de **messages** :

```
{ clé, params: { <nom>: <valeur>, ... }, niveau: erreur | avertissement | information }
```

- **clé** : une clé de la famille `message`, résolue par langue.
- **params** : valeurs **nommées** des placeholders.
- **niveau** : niveau **utilisateur** (fixé par le script — il connaît la gravité) ; il conditionne le
  comportement (erreur bloque, avertissement passe, information indique) et se décline multi-canal.
- Ce niveau **utilisateur** n'a **rien à voir** avec le **niveau de log technique** (debug/info/warn…
  façon log4j), qui relève d'un **ADR journalisation/observabilité distinct** (réservé) : le log
  technique est destiné à l'exploitant, pas à l'utilisateur, et n'est pas forcément traduit.

## Alternatives écartées

- **Texte en dur** (hors clé) : impossible à traduire. Écarté.
- **Le code technique EST la traduction** : soude l'identifiant (immuable) au contenu (variable, à
  plusieurs textes par objet). Remplacé par « le code est la **racine** de l'espace de clés ».
- **Préfixe `u_` réservé à Fabrica / espace nu au projet** (façon inverse) : réserverait un seul
  préfixe à Fabrica (trop étroit pour ses futurs modules) et masquerait ce qui est custom. Remplacé
  par « préfixe obligatoire pour tous + liste de préfixes réservés Fabrica + `u_` met en avant le
  custom ».
- **Partage de listes entre attributs** : les listes de deux attributs divergent souvent (ex. statuts
  d'objets différents) ; le partage créerait un couplage artificiel. Une **liste par attribut**,
  dupliquée si besoin.
- **Placeholders positionnels** : fragiles au réordonnancement multilingue. Remplacés par nommés.

## Renvois / dettes ouvertes

- **Points d'entrée / scripts** (ADR à venir) : appliqueront le *contrat de message* ci-dessus.
- **Journalisation / observabilité** (ADR à venir) : logs techniques, niveau courant réglable au
  runtime, affinage par composant (façon log4j).
- **Cycle de vie des objets** (ADR à venir) : états et transitions (soulevé par le cas des statuts).
- **Surcharge des traductions** : extension additive réservée.
