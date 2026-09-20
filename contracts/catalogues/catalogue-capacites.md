# Catalogue — Capacités d'entité

> Registre des capacités d'entité. Mécanisme : **ADR-0025**. Motif catalogue : constitution
> (« Catalogues »). Règle : une capacité qui **porte une décision** a un ADR (renvoi) ; sinon elle
> est **décrite** ici. Ce fichier ne contient jamais de décision — il renvoie ou décrit.
> Destiné à devenir une **donnée système réflexive**. Source de vérité : ce fichier (git).

## Système (présentes sur toute entité, non supprimables ; réservent leurs noms ; comportement configuré)

| Capacité | Apporte | Décision |
|---|---|---|
| `colonnes-systeme` | `sys_id`, dates, `_by`, `sys_version`, `sys_class` | **ADR-0020** |
| `actif` | booléen `actif` (suppression logique, filtre) | **ADR-0023** |
| `valeur-affichage` | représentation d'affichage d'un objet (*display value*) — consommée par références, listes, historisation | **ADR-0027** (déf.) ; dette « attributs calculés » |
| `audit` | traçabilité sécurité — **comportement configuré par le niveau de traçabilité** de l'entité/attribut (rien par défaut) | **ADR-0028** |

## Optionnelles (activées par le projet)

| Capacité | Apporte | Décision |
|---|---|---|
| `à-états` | `state` + graphe de transitions + cycle de vie | **ADR-0019** |
| `numérotée` | `number` + préfixe + séquence + unicité | *décrite ci-dessous (ADR à créer si besoin)* |
| `historiser` | historique des changements « qui a changé quoi », par attribut marqué | **ADR-0027** |

### `numérotée` (description — voir dette « numérotation »)
Active un identifiant fonctionnel `number` : n'existe que si activée ; alors **obligatoire et
unique**. Paramétrage : préfixe (trigramme/quadrigramme, immuable au démarrage) + nombre de chiffres
(6 par défaut). Numéros croissants, uniques, **sans garantie de continuité** (réservation anticipée →
trous acceptés). Compteur du prochain numéro stocké par Fabrica. *Porte des décisions (préfixe
immuable, séquence, continuité) → mérite un ADR propre à terme.*

## Pressenties (identifiées, non spécifiées)

| Capacité | Intention | Lien |
|---|---|---|
| `journalisée` | logs techniques attachés | dette journalisation |
| `affectable` | affectation à personne/groupe, file de travail | dette affectation (version ultérieure) |
| `commentable` | notes/commentaires attachés | dette commentaires (axe ligne) |
| `hiérarchique` | parent de même type (arborescence) | à spécifier |

> Les pressenties ne sont **pas** des décisions actées — elles marquent des capacités attendues,
> à spécifier (ADR ou description) quand elles seront traitées.
