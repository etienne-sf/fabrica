# ADR-0028 — Audit de traçabilité (« T » de DICT)

**Statut :** Proposé — 2026-07-11
**Portée :** décision du **cœur** (Fabrica), générique (Principe IV). Réalise la **Traçabilité** du
cadre DICT (constitution). Distinct de l'**historisation** (métier, exhaustive — ADR-0027) et de la
**journalisation technique** (exploitants — ADR distinct). À éclater en
`contracts/adr/0028-audit-tracabilite.md`.

---

## Contexte

L'audit trace les **accès et actions sensibles** pour la **conformité** (public : sécurité, pas
métier). Il recoupe l'historisation sur la **capture** (un changement peut être à la fois historisé
et audité) mais en diffère par la **finalité** : l'historisation dit *ce qui a changé* (choisi par
le **métier**) ; l'audit dit *qui a fait quoi de sensible* (choisi par la **sécurité**). Plusieurs
exigences déjà posées relèvent de l'audit : impersonation (ADR-0012), break-glass (ADR-0010/0012),
peupler un groupe (ADR-0012).

## Décision — l'audit est intégralement du paramétrage projet

**Fabrica ne définit AUCUNE politique d'audit** — y compris sur ses **tables système**. La
traçabilité relève de la **politique de l'entreprise** ; Fabrica ne peut pas la présumer. Fabrica
fournit le **mécanisme** (les niveaux, la capture) ; le **projet** définit **tout** : le **sens des
niveaux** *et* leur **affectation**.

Conséquence assumée : même les tables sensibles de Fabrica (rôles, ACL…) **ne sont pas auditées par
défaut** — c'est au projet de l'activer s'il le veut.

## Décision — l'audit est une capacité système, configurée par le niveau de traçabilité

`audit` est une **capacité système** (ADR-0025) : **présente sur toute entité**, comme un organe
dormant. **Déclarer (ou supprimer) un niveau de traçabilité** sur une entité ou un attribut ne crée
ni ne supprime la capacité — il **règle son comportement à l'exécution** : à « aucun » elle ne fait
rien ; à T2/T3/T4 elle capture selon le niveau. Le projet ne gère jamais « la capacité audit » — il
déclare des **niveaux de traçabilité**, et la capacité système `audit` s'exécute en conséquence.

## Décision — les 4 niveaux de traçabilité (définis par le projet)

- Le **niveau de traçabilité** est une **propriété de gouvernance de premier rang** de l'entité,
  surchargeable par attribut (motif entité→attribut, ADR-0012). Il a du sens **en soi** (il exprime
  une politique), indépendamment du mécanisme de capture — d'autres critères DICT (chiffrement…)
  pourront un jour être déclarés de la même façon. Étant déclaré et lisible, il **contribue à la
  mesurabilité DICT** (constitution) : « quel est le niveau de traçabilité de cette donnée ? » se lit
  dans le métamodèle réflexif.
- Le **sens des 4 niveaux** est **défini par le projet** (il dépend de sa politique).
- **Affectation** : un **niveau par entité** (grain grossier), **surchargeable par attribut** (à la
  hausse ou à la baisse).

## Décision — configuration par défaut fournie mais NON appliquée

Fabrica **propose une configuration par défaut** (une gradation type : *aucun / écritures /
+ actions sensibles / + lectures*), **disponible mais non appliquée**. Le projet peut :
- l'**ignorer** (définir la sienne de zéro) ;
- l'**appliquer comme base** de départ, puis ajuster ;
- l'**appliquer en écrasant** sa configuration existante.

**Découplage total (pas de merge)** : Fabrica peut faire **évoluer** son défaut sans changer ce que
le projet a appliqué (le défaut n'est jamais actif) ; le projet n'adopte un nouveau défaut que par
**acte explicite** (écrasement volontaire). Le gabarit Fabrica et la copie projet sont **deux
exemplaires distincts** — le gabarit évolue avec Fabrica, la copie n'évolue que sur décision projet.
*(Ce mécanisme « gabarit fourni mais non appliqué, copié sur décision projet » est noté comme
**patron à formaliser** au second usage — dette.)*

## Décision — inaltérabilité (double)

- **Configuration d'audit inaltérable** : elle fait partie du **métamodèle du projet** (livrée,
  versionnée, changée par **déploiement avec tests** — comme le graphe de transitions, ADR-0019).
  Pas de modification à chaud de « ce qui est audité ».
- **Données d'audit inaltérables** : **append-only strict** — personne ne modifie ni ne supprime un
  événement d'audit (DELETE/UPDATE révoqués, y compris au compte applicatif). L'accès base direct
  reste le risque d'exploitation assumé (ADR-0008). Sinon l'audit ne prouve rien.

## Décision — capture

- **Écritures** : **capture partagée avec l'historisation** — le même trigger (after, ADR-0027)
  alimente l'historique (si l'attribut est historisé) **et** l'audit (selon le niveau de traçabilité).
  Un mécanisme, deux destinations selon les politiques. Pas de duplication.
- **Lectures** : PostgreSQL **ne déclenche pas de trigger sur SELECT** — auditer les lectures ne peut
  donc pas passer par trigger. Au démarrage : **capture par la couche Fabrica** (le plus simple ;
  quand elle sert une lecture auditée, elle enregistre l'événement). Défaut assumé : rate les lectures
  faites en accès base direct (risque d'exploitation, ADR-0008).
- **Transparence pour le projet** : le projet déclare un **niveau de traçabilité**, jamais un
  mécanisme. Le mécanisme de capture des lectures (couche Fabrica → pgAudit → CDC) peut donc **changer
  de façon transparente** pour le projet (isolation projet/implémentation, ADR-0007).

## Hors périmètre (renvois)

- **Externalisation** (SIEM, puits externe) : **optionnelle, non implémentée**. À terme via CDC /
  mécanismes post-modification, au niveau infrastructure.
- **Audit par ligne** : réservé (rejoint la complexité de l'axe ligne, ADR-0026).
- **Patron « gabarit fourni non appliqué »** : dette (à formaliser au second usage).
- **Journalisation technique** : ADR distinct.

## Alternatives écartées

- **Fabrica définit une politique d'audit par défaut appliquée** : imposerait une politique que le
  client n'a pas choisie, et créerait un merge à la montée de version. Remplacé par « défaut fourni
  mais non appliqué, copié sur décision projet ».
- **Mécanisme de capture d'écriture séparé de l'historisation** : duplication inutile. Capture
  partagée.
- **Audit des lectures par trigger** : impossible (pas de trigger sur SELECT). Capture couche Fabrica.
- **Données d'audit modifiables** : ne prouveraient rien. Append-only strict.
