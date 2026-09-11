# ADR-0012 — Modèle d'autorisation : rôles, ACL, groupes et administration

**Statut :** Proposé — 2026-07-11
**Portée :** décision du **cœur** (Fabrica), générique (Principe IV). Met en œuvre le principe
d'autorisation par (entité, attribut) de la **constitution** (« Discipline de développement »),
qu'il **référence sans le redédire**. Présupposé par les ADR 0003, 0008, 0009, 0010. À éclater
en `contracts/adr/0012-autorisation.md`.

> Numérotation = identifiant chronologique. Cet ADR est logiquement antérieur à plusieurs qui
> le référencent ; l'ordre de lecture est porté par l'index, pas par le numéro.

---

## Modèle d'ensemble : deux axes, pilotés par les rôles

L'autorisation se compose de **deux axes** :
- **Axe structurel — entité/attribut** (le présent ADR) : quel *type* d'objet, quelles
  *colonnes*. L'entité est le grain grossier ; l'attribut en est le raffinement (droit
  d'attribut hérité de l'entité par défaut) — **pas** un axe indépendant.
- **Axe des lignes** (ADR distinct à venir) : quelles *instances*. Réalisé par la RLS réservée
  de l'ADR-0010.

**Les deux axes sont pilotés par la même structure de rôles.**

**Invariant de conjonction :** le droit effectif sur une donnée = **conjonction** de l'axe
structurel ET de l'axe ligne — jamais l'union. L'axe ligne ne peut que *restreindre*.

## Modèle additif pur

- **À l'intérieur d'un axe, les ACL s'additionnent** : le droit effectif est l'**union** des
  droits accordés. **Aucune interdiction explicite (deny)**, aucun ordre de résolution à définir.
- La **restriction** se fait par conjonction entre axes (structurel ET ligne) et par
  **intersection** (impersonation) — jamais par soustraction dans un axe.
- Rationale : union et intersection sont commutatives → résolution **déterministe**. Prix
  assumé : « tous sauf untel » se modélise en n'incluant pas untel, pas par un deny.

## Rôles et affectation

On n'affecte **jamais** d'ACL élémentaires directement à un utilisateur (voir alternatives). Les
droits sont portés par des **rôles** :
- un rôle porte des droits via des **ACL** (voir plus bas) ;
- les rôles se **composent en rôles parents, sans limite de niveau** (hiérarchie de rôles) ;
- un rôle s'affecte à un **utilisateur** ou à un **groupe**. **Recommandation de passage à
  l'échelle : privilégier les groupes** (chemin uniforme, plus traçable) ; l'affectation directe
  à un utilisateur reste possible car pratique.
- un rôle de projet **peut hériter** d'un rôle standard de Fabrica (composition par-dessus, le
  livré restant intact — ADR-0013). Pas de distinction « rôle-brique / rôle-attribuable » : tous
  de même nature ; la présentation agrégée relève de l'IHM d'administration (sujet distinct).

## ACL : forme, natures, charge utile

L'**ACL élémentaire** a la forme **(rôle, nature, charge utile)**. Il n'y a **pas** de champ
« niveau » commun : ce qu'on appelait « niveau » est remonté au rang de **nature**, pour la
lisibilité (on lit `donnée:lecture`, pas « accès-donnée, variante lecture »).

**Convention de nommage des natures : `famille:déclinaison`.** La famille nomme le domaine du
droit, la déclinaison le degré ou l'action. Rend le catalogue lisible à mesure qu'il grandit.

**Catalogue de natures : fermé et extensible**, propriété de Fabrica. Une nature n'existe que si
Fabrica sait la **résoudre, l'imposer et l'intersecter** (critère d'admission). Un projet
*référence* une nature, il n'en *invente* pas. Ajouter une nature = étendre aussi la résolution
et l'intersection (coût de mécanique, ce qui justifie la fermeture).

**Charge utile structurée par nature** : les informations varient selon la nature. La table
porte le commun (rôle, nature) + une charge utile dont le schéma dépend de la nature ;
**validation portée par Fabrica**. Évite l'explosion de colonnes. **La cible est toujours un
identifiant désigné, jamais une donnée dupliquée** : un code d'attribut (reconnu par le
métamodèle), un nom de fonctionnalité (reconnu par le *code* de Fabrica), une référence de
condition de ligne (reconnue par le catalogue).

**Natures « donnée » (charge utile = cible seule), ordre croissant de puissance :**
1. `donnée:lecture`
2. `donnée:lecture_écriture`
3. `donnée:lecture_écriture_suppression`

Cet ordre est à la fois l'**ordre d'affichage** (du moins au plus permissif) et l'**ordre de
résolution** connu de Fabrica : l'additif prend le plus fort (union de `lecture` et
`lecture_écriture_suppression` = `lecture_écriture_suppression`) ; l'héritage entité→attribut est
la règle « faute de droit propre à l'attribut, appliquer celui de l'entité » (aucun champ ordinal
à propager). On évite `full`, non explicite.

**Nature « fonctionnalité » (ex. `fonctionnalité:exécuter`, charge utile = nom d'une
fonctionnalité livrée par Fabrica)** : droit d'exécuter. **Se combine avec les droits `donnée:`,
ne les court-circuite JAMAIS** — « exporter » donne le droit d'utiliser l'outil ; ce qui est
exporté reste borné par les droits de l'utilisateur (entité/attribut/ligne). Empêche l'élévation
par fonctionnalité (cohérent ADR-0009). Les fonctionnalités « pure action » (sans cible de
donnée) portent seulement le droit d'exécuter.

*(Familles futures — API, rapport… — suivront la même convention `famille:déclinaison`, avec
leurs propres déclinaisons ordonnées si pertinent.)*

## Défauts et attributs système

- **Défaut d'un attribut** : hérite du droit de l'**entité** mère (règle, pas champ propagé).
- **Défaut d'un utilisateur** : **aucun droit** (deny par défaut).
- **Admin (défaut)** : `donnée:lecture_écriture_suppression` partout, **sauf** les attributs
  système.
- **Utilisateur (défaut)** : ne lit rien.
- **Attributs système** (`id`, dates, utilisateur ayant modifié, **classe réelle de la ligne**…)
  : **posés par le système, jamais écrits par l'utilisateur** — hors du modèle de droits
  utilisateur. Lisibles seulement si l'entité l'est. **La liste normative des attributs système
  est définie dans l'ADR « colonnes système » (à créer)** ; le présent ADR n'en donne que des
  exemples.

## Groupes et gestion des appartenances

- Un **groupe** rattache une population à des rôles. **Rattachement par règle** (appartenance
  dérivée d'attributs de l'utilisateur) : **réservé, non construit** (frontière calée).
- **Gérer un groupe = gérer ses membres, JAMAIS les rôles qu'il porte** (modifier les rôles
  supposerait de connaître les catalogues et rouvrirait l'élévation — ADR-0013).
- **Cas standard** : un membre est **marqué gestionnaire** (attribut sur l'appartenance). Étant
  membre, il a déjà les droits du groupe → **aucune élévation possible par construction**. Il
  ajoute/retire des membres, et **ajoute/retire des gestionnaires (symétrique)** — auto-propagation
  assumée, au service de l'autonomie des équipes.
- **Cas séparation des pouvoirs** : un groupe peut être **sans gestionnaire interne** ; un
  **autre groupe** porte le droit d'administrer ses appartenances. Le gestionnaire n'est alors
  pas membre et ne bénéficie pas des droits → *gérer* dissocié de *bénéficier*. Toujours via un
  groupe (jamais un droit collé à un individu).
- **Cardinalité** : « gère les membres de » est **un-à-plusieurs** (un groupe gestionnaire gère
  plusieurs groupes gérés) ; **un seul groupe gestionnaire par groupe géré**. Autorise une
  **pyramide de gestion** (hiérarchie de délégation), **distincte de la hiérarchie de rôles** :
  un groupe peut être haut en gestion sans être puissant en droits (un « super-gestionnaire »
  distribue sans détenir).
- **Invariant d'enracinement** : toute chaîne de délégation remonte à l'administrateur en un
  nombre fini d'étapes ; **pas de cycle** (arbre/graphe acyclique enraciné sur l'admin). Le
  **sommet est adossé au rôle admin** (super-gestionnaires = détenteurs du rôle admin, ou groupe
  peuplé automatiquement depuis eux), ce qui ferme la récursion.
- **Garde résiduelle** : peupler un groupe (surtout puissant) est un droit de **gouvernance
  sensible** → **audité**.

## Impersonation

Droit réservé à un **administrateur** : prendre le rôle d'un autre utilisateur.
- **Droits = intersection des droits effectifs *résolus*** de l'admin ET de la cible (chaque
  hiérarchie aplatie en droits concrets, **puis** intersectée — jamais intersection des rôles).
  Un admin ne gagne ni ne prête aucun droit.
- **Traçabilité** : accès en impersonation tracés (identité réelle + empruntée), a minima dans
  les logs ; la trace dans l'historique de l'objet dépend de l'ADR historisation/traçabilité.

## Garantie d'administration (jamais de verrouillage)

1. **Prévention** : refuser toute opération (retrait d'affectation, désactivation/suppression de
   compte) qui laisserait **zéro** affectation active du rôle admin.
2. **Existence** : le **rôle** admin standard est **non supprimable**. Le **compte** admin par
   défaut est **désactivable** (mot de passe vidé) **une fois un autre admin actif** — satisfait
   l'interdiction des comptes génériques sans violer l'additif (on désactive un *compte*, on ne
   soustrait pas un *droit*). Distinction rôle/compte : le rôle est permanent, les comptes gérables.
3. **Récupération inconditionnelle** : il doit **toujours** exister un moyen, **indépendant de
   l'application**, de restaurer un accès d'administration (« un blocage ne doit jamais être
   irréparable »). *Mécanisme* → ADR « données système / montées de version » : procédure SQL
   directe fournie par Fabrica, rejouant la **définition courante** du rôle admin (pas une copie
   figée), insérant un utilisateur de secours via une **capacité d'authentification locale**
   (ADR-0010), sans écraser les données projet, exécutable Fabrica arrêtée, auditée.

## Hors périmètre (renvois)

- **Axe des lignes / RLS** : ADR distinct (raccord ADR-0010 ; invariant « ne jamais outrepasser
  l'entité »). Y vivent les **notes/commentaires** (droit dérivé du droit sur l'entité commentée ;
  table commentaire de Fabrica portant l'attribut système `classe`, filtrée sur les classes
  lisibles) et la **délégation de gestion** exprimée comme ACL de ligne sur les appartenances.
- **Colonnes système** : ADR distinct (liste normative, dont la classe réelle de la ligne).
- **Historisation / traçabilité** : ADR distinct (mécanisme uniforme pour tout objet, Fabrica ou
  projet).
- **IHM d'administration des rôles** ; **regroupement des entités en domaines** : sujets distincts.

## Alternatives écartées

- **Affecter des ACL élémentaires directement à un utilisateur, hors de tout rôle** (ingérable) —
  à ne pas confondre avec affecter un *rôle* à un utilisateur, qui reste permis.
- **Deny explicites** (complexité + ordre de résolution).
- **Entité et attribut comme deux axes indépendants** (l'attribut est le raffinement de l'entité).
- **Deux systèmes de rôles distincts** pour structurel et ligne.
- **Un champ « niveau » commun à toutes les natures** (ne concerne que les données ; remonté au
  rang de natures `donnée:*` pour la lisibilité).
- **Une colonne d'ACL par information** (explosion de colonnes ; remplacée par la charge utile).
- **Une table « fonctionnalités » redondante** (le nom suffit, reconnu par le code).
- **Règle de protection seulement en GraphQL, pas en base** (contournable — ADR-0008).

## Conséquences

- La **résolution des droits effectifs** (aplatir hiérarchie de rôles + ACL + défauts en droits
  concrets) est une brique nécessaire, consommée par la frontière de mutation et l'impersonation.
- ACL de projet et ACL livrée par Fabrica **cohabitent additivement** (ADR-0013).
- Ce modèle est un **pilier** du cœur, probablement à éclater en plusieurs specs à la construction.
