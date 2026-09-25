# TOOL-0002 — Revue de montée de version

**Statut :** Proposé — 2026-07-11
**Famille :** **Outillage** (`tool/`). Cycle de vie du projet, phase **packaging** (tool-0001).
Générique (Principe IV). **Versant produit :** l'écran de revue est rendu par l'IHM générique
(ADR product-0030) — auto-application.

---

## Contexte

Quand Fabrica monte de version (en **environnement de développement**), certaines évolutions
**impactent le projet** : il doit faire quelque chose (compléter une valeur devenue obligatoire),
ou du moins en être informé (nouvelle capacité, nouveau crochet). La **Revue de montée de version**
présente ces impacts et conditionne le passage en RUN. Traiter les impacts **montre l'adaptation
progressive du projet à la nouvelle version** — d'où le nom (« revue », pas « dette » : les entrées
ne sont pas toutes des dettes).

## Décision — un mécanisme, pas une liste (extensible par construction)

Cet ADR définit le **mécanisme** (comment un impact est détecté, présenté, fermé, et comment il
bloque le packaging), **pas** l'énumération des impacts possibles. Chaque évolution future de Fabrica
qui crée un impact **déclare une entrée** dans ce mécanisme, **sans amender cet ADR**. Le sujet
grossit en *contenu* (plus de types d'impacts), jamais en *mécanisme*. (Motif « catalogue fermé,
extensible par Fabrica » appliqué aux impacts — constitution.)

## Décision — deux niveaux au MVP (warning réservé)

- **Bloquant** : le **packaging échoue** tant qu'au moins un impact bloquant est **ouvert**.
- **Info** : signalé, ne bloque pas.
- **Warning : réservé (post-MVP).** Il tire un sujet lourd — il faudrait, **par version**, qu'un
  responsable **qualifie** chaque évolution non bloquante en info ou warning (curation par version),
  et gérer une **confirmation forcée** au packaging. Non MVP.

## Décision — fermeture : bloquant = auto-vérifiable, info = acquittement

- **Bloquant ⇒ auto-vérifiable.** Fabrica **détecte** le problème structurel (valeur manquante,
  référence cassée, usage d'un élément disparu), **montre la liste des occurrences** et **l'action à
  faire**, et **ferme la dette automatiquement** dès que c'est résolu. Aucun acquittement humain — le
  fait *est* la résolution. Principe : *dès que possible, Fabrica détecte seule*, et alors elle
  **indique clairement ce que le développeur doit faire**.
- **Info ⇒ acquittement.** Rien à vérifier (opportunité, changement de comportement à comprendre) :
  le développeur **acquitte** (« vu »). Les entrées **acquittées sont masquées par défaut**, mais
  **toujours ré-accessibles**.

## Décision — tableau de bord de la revue (MVP)

L'écran de **Revue de montée de version** est **MVP** : tester le mécanisme de montée de version est
un **objectif explicite du MVP**, et l'on ne peut le tester sans l'écran qui montre les impacts et
permet de les traiter. Il liste les impacts (bloquants ouverts, infos non acquittées), leur statut,
et le détail (pour un bloquant : la liste des entités/attributs concernés). Il est réalisé via
**l'IHM générique** (ADR-0030) — les impacts sont des données ; Fabrica les gère comme telles
(auto-application), donc l'écran coûte peu.

## Auto-détection par type d'impact (première analyse)

| Impact | Auto-détecté ? | Niveau | Ce que Fabrica montre |
|---|---|---|---|
| Nouvel attribut de métamodèle **obligatoire** | **Oui** | Bloquant | Liste des entités/attributs sans valeur |
| Champ de **format** devenu obligatoire (tool-0001) | **Oui** | Bloquant | Occurrences non remplies |
| **Référence cassée** (report, ACL, effet… visant un élément disparu/modifié) | **Oui** | Bloquant | Les usages à corriger |
| Nouvel attribut de métamodèle **optionnel** | Oui (existence) | Info | Signale ; le projet liste lui-même les entités s'il veut renseigner |
| Nouvelle **capacité** livrée | Partiel (existence) | Info | Signale la capacité disponible |
| Nouveau **crochet / point d'extension** | Non (usage) | Info | Signale le crochet disponible |
| **Changement de comportement** (une règle Fabrica évolue) | Non | Info | Décrit le changement |

Motif : Fabrica **auto-détecte et auto-ferme** tout ce qui est une **vérification structurelle sur le
métamodèle** (valeur manquante, référence cassée) → **bloquant, auto-vérifiable**. Ce qui relève d'un
**jugement/opportunité** → **info, à acquitter**.

## Décision — rôle

Au MVP, un seul rôle dans l'outil : **Développeur**. Le **système de rôles de l'outil de
développement** (2ᵉ système d'autorisation, distinct du RUN — ADR Produit-0012) est **réservé**.

## Hors périmètre (renvois / réservé)

- **Warning** + qualification info/warning **par version** : post-MVP.
- **Lien direct** depuis un impact vers le **formulaire de correction** de l'entité : post-MVP —
  retenu comme **premier cas de test de l'évolution incrémentale** (ajout purement additif, isolé, à
  faible enjeu : valide la capacité de la chaîne Spec Kit + génération à évoluer sans casse ; l'IHM
  étant un interprète dynamique — ADR-0030 — cet ajout ne régénère rien).
- **Rôles de l'outil de développement** : réservé.

## Alternatives écartées

- **Énumérer les impacts dans l'ADR** : imposerait de l'amender à chaque évolution. Mécanisme, pas
  liste.
- **Acquittement des bloquants** : inutile — ils se ferment par les faits (auto-vérifiables).
- **Auto-détection de la résolution des infos** : rien à résoudre (prise d'acte). Acquittement.
- **Tout inclure au MVP par peur du re-work** : nierait l'incrémental, qui est justement ce que le
  MVP doit tester. Le lien de correction reste post-MVP comme cobaye de cet incrémental.
