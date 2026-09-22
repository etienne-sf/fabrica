# ADR-0016 — Moteur des effets : orchestration, point fixe, application

**Statut :** Proposé — 2026-07-11
**Portée :** décision du **cœur** (Fabrica), générique (Principe IV). Orchestre le moteur de règles
(ADR-0015) et applique les effets (ADR-0006). À éclater en `contracts/adr/0016-moteur-des-effets.md`.

---

## Contexte

Les effets (ADR-0006) sont déclenchés par des règles, dont l'évaluation doit converger (une règle
peut modifier une donnée que d'autres règles lisent). L'analyse a montré que **le point fixe
n'appartient pas au moteur de règles** — un moteur de règles générique est sans état et sans effet,
il fait *une passe* d'évaluation (ADR-0015). Le point fixe et l'application des effets relèvent d'un
composant propre à Fabrica : le **moteur des effets**.

## Trois pièces distinctes

- **Le déclencheur** : ce qui initie une évaluation — un champ modifié en formulaire, une mutation
  API, un clic de modification en liste, une ingestion. Il fournit l'état de départ + la
  modification demandée. **Le déclenchement fondamental est toute tentative de modification de
  donnée** (ADR-0006).
- **Le moteur de règles** (ADR-0015 ; agnostique, standard, possiblement une bibliothèque externe) :
  **une passe**. Entrée = un état + les règles. Sortie = les conclusions (effets à poser,
  modifications de données à faire). **Sans état, sans effet, sans boucle.** Encapsulé et remplaçable.
- **Le moteur des effets** (propre à Fabrica) : l'**orchestrateur**. Il tient la copie de travail,
  appelle le moteur de règles, applique les modifications de données conclues, re-boucle jusqu'à
  stabilité, détecte la non-convergence et les contradictions, puis applique l'état final selon le
  régime. **C'est lui qui porte le point fixe.**

## Décision — la boucle jusqu'au point fixe

Le moteur des effets travaille sur une **copie de travail** de l'état (rien n'est validé tant que la
convergence n'est pas atteinte — cohérent transactionnalité) :

1. Appelle le moteur de règles (ADR-0015) sur l'état courant → reçoit des conclusions.
2. **Distingue deux natures de conclusions :**
   - **modifications de données** (ex. `vider` un champ, cascade « fermer la tâche ferme le
     ticket ») → appliquées à la copie de travail, et **relancent** une passe (l'état a changé) ;
   - **effets non-modificateurs** (`interdire_modification`, `rendre_obligatoire`, `masquer`) →
     **accumulés**, ils ne relancent PAS la boucle (ils ne changent pas l'état de données).
3. **Critère d'arrêt** : la boucle re-tourne **tant qu'une passe produit une modification de
   données** ; elle s'arrête dès qu'une passe n'en produit aucune. (Les règles ne se déclenchent que
   sur modification de données ; un champ qui devient obligatoire ou non-modifiable n'est **pas** une
   modification de données.)
4. À la convergence, **applique l'état final** (les données modifiées + les effets accumulés) selon
   le régime (projection formulaire / vérification donnée, ADR-0006).

Ce critère garantit la **terminaison** : seules les modifications de données prolongent l'itération,
et le socle du métamodèle borne ce qui peut être modifié.

## Décision — non-convergence

La détection de non-convergence appartient au **moteur des effets** (seul à voir la boucle ; le
moteur de règles fait des passes indépendantes et ignore qu'il est rappelé). Deux règles qui se
défont (A vide un champ, B le remplit) ne convergent pas → le moteur des effets **détecte**
(nombre maximal d'itérations et/ou réapparition d'un état déjà vu) et produit une **erreur propre et
tracée** (« les règles ne convergent pas »). Il ne tourne jamais sans fin.

## Décision — contradictions d'effets

Fabrica **ne résout pas** les contradictions de règles (pas de préséance arbitraire qui masquerait le
défaut) : elle les **signale et trace** comme un **défaut du jeu de règles du projet, à corriger**.
Un jeu de règles contradictoire est un jeu de règles faux (Principe II : le mécanisme ne devine pas,
il rend visible l'erreur). Exemple : à la convergence, un champ à la fois `masquer` et
`rendre_obligatoire`.

**Compromis de détection (où placer l'effort) :**
- **À la validation (statique, peu coûteux)** : les contradictions **structurelles** — celles qui se
  vérifient en **comparant des déclarations entre elles et avec le socle** du métamodèle, sans
  raisonner sur des états de données. Portent sur les règles **déclaratives** uniquement.
- **Au runtime (moteur des effets)** : les contradictions **conditionnelles** (dépendantes de l'état,
  ex. « masquer si A » + « obligatoire si B » quand A et B sont vrais ensemble) et **toutes** celles
  impliquant un **script**. Détectées quand elles surviennent, erreur tracée.
- **Borne explicite** : on ne fait **jamais** d'analyse d'atteignabilité d'états (indécidable, coût
  explosif) ni d'analyse statique des scripts (impossible par nature). C'est ce qui empêche le sujet
  de devenir « trop compliqué ».
- **Justification** : le runtime attrape de toute façon tout ce que le statique raterait. La
  validation statique n'est donc pas un filet (le runtime l'est) mais un **confort** de détection
  précoce, dimensionné en conséquence — beaucoup gagné sur les erreurs grossières, rien tenté sur
  l'indécidable.

## Anti-clignotement (induit)

Puisque le moteur des effets n'**applique** l'état qu'après convergence, l'écran ne voit que l'**état
final**, jamais les états intermédiaires de l'itération → **aucun clignotement** (les champs ne
réapparaissent pas puis se re-masquent…). Propriété **induite** par « on calcule entièrement, puis on
applique » ; à préserver (ne jamais appliquer d'état intermédiaire).

## Périmètre (ce que cet ADR ne traite PAS)

- Le **catalogue d'effets** et leurs régimes d'application → ADR-0006.
- Comment une **règle est écrite/fournie** (déclarative ou script, mise à disposition, contrat) →
  ADR points d'entrée / scripts.
- Comment le **moteur de règles évalue** en interne (inférence, une passe) → ADR-0015.
- Les **messages** (clés + placeholders) → ADR traduction.

## Conséquences

- Le moteur de règles reste **remplaçable** : toute l'intelligence propre à Fabrica (boucle, copie de
  travail, application, convergence) vit dans le moteur des effets ; le moteur de règles (ADR-0015) est
  une commodité interchangeable derrière un contrat mince (une passe : état + règles → conclusions).
- La **copie de travail** a un coût (matérialiser l'état en cours de convergence) assumé au profit de
  la transactionnalité (rien n'est validé si ça ne converge pas).
- Les **contradictions** relèvent de la responsabilité du projet (jeu de règles à corriger), pas d'un
  rattrapage silencieux de Fabrica.
