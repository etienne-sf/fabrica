# ADR-0035 — Reporting v1

**Statut :** Proposé — 2026-07-11
**Famille :** **Produit** (`product/`). Dernier sujet fonctionnel du MVP. Générique (Principe IV).
**Versant tool :** éditeur de rapports (composer un rapport / un tableau de bord) — dette.

---

## Contexte

La démonstration du référentiel EA doit montrer la **maîtrise du patrimoine**, au moins par une
première version du reporting. Le reporting est un gros sujet (ServiceNow en a fait un module séparé) :
la v1 est volontairement étroite. L'architecture d'exécution (requête directe, pré-agrégation, moteur
analytique…) **n'est pas préjugée** — elle relève de la phase technique.

## Décision — un rapport est du métamodèle (v1)

- En v1, les rapports et tableaux de bord sont **définis par le projet**, dans le dépôt
  (`reports/`, tool-0003), **versionnés** et **immuables en production** — comme les vues.
- Un rapport qui référence un élément disparu ou modifié (attribut, valeur de liste) est une **dette
  bloquante auto-détectée** de la revue de montée de version (tool-0002).
- **Réservé** : **rapports créés par les utilisateurs**, éventuellement **en dupliquant** un rapport du
  métamodèle. C'est le patron **« gabarit fourni »** (constitution) : copie sur décision, devenue
  propriété de l'utilisateur, sans lien de fusion avec l'original. Les rapports utilisateur seront alors
  des **données de production**.

## Décision — périmètre v1

Un **rapport** se compose de :
- une **entité source** ;
- des **filtres** (réutilisant les conditions déclaratives existantes) ;
- un **regroupement optionnel**, sur **un seul** attribut (typiquement une liste de valeurs ou une
  référence — « applications par capacité », « par statut ») ;
- une **mesure** : **comptage** ; **somme** ou **moyenne** d'un attribut numérique ;
- une **visualisation** : **tableau**, **diagramme en barres**, **camembert**.

Un **tableau de bord** est un **assemblage de plusieurs rapports** sur une page.

Mesures et visualisations forment des **catalogues fermés** (`catalogue-rapports.md`), extensibles par
Fabrica.

## Décision — le rapport ne contourne jamais l'autorisation

- Un rapport **ne manipule que les données que l'utilisateur a le droit de voir** : ses requêtes passent
  par la même frontière que le reste (API, RLS en base — ADR-0008, 0026). Un comptage ne porte que sur
  les lignes visibles.
- On ne peut **filtrer ni regrouper** sur un attribut qu'on n'a pas le droit de lire (filtrer découle de
  lire — ADR-0012).
- Conséquence : **le même rapport peut afficher des chiffres différents selon qui le consulte**. Au MVP,
  c'est **accepté sans signalement**. Hors MVP, c'est **impératif** à traiter (risque de décrédibiliser
  Fabrica) — voir ci-dessous.

## Décision — performance : tolérance MVP

Au MVP, il est **accepté qu'une analyse lourde dégrade ou écroule la plateforme** (les bases de données
résistent en général correctement). L'exigence de protection (ADR-0029, journalisation) **revient
post-MVP** : délais maximaux, limites de volume, isolation, ou toute autre mesure décidée en phase
technique.

## Hors périmètre (renvois / dettes)

- **Signalement des conditions d'exécution** (post-MVP, **impératif**) : un en-tête indiquant que le
  rapport est calculé sur le périmètre d'accès de l'utilisateur. **Sans fuite** : on ne compte **pas**
  les lignes cachées (ce comptage révélerait leur existence et leur nombre) ; on déduit **des règles**
  (droits effectifs, ADR-0012) si le périmètre est **potentiellement restreint** — une condition de
  ligne sur l'entité suffit à le savoir, sans toucher aux données. Formulation claire et synthétique à
  concevoir.
- **Rapports créés par les utilisateurs** (duplication d'un rapport du métamodèle — gabarit fourni).
- **Regroupements à plusieurs niveaux** ; **rapports traversant des relations** ; **rapports programmés
  ou exportés**.
- **Protections de performance** (post-MVP).
- **Éditeur de rapports** (versant tool).
- Choix techniques : moteur d'agrégation, bibliothèque de graphiques (React, *buy*) — phase technique.

## Alternatives écartées

- **Rapports comme données dès la v1** (seed + création utilisateur) : exige une IHM de création et pose
  la mise à jour d'un rapport livré puis modifié en production. Métamodèle en v1, création utilisateur
  réservée.
- **Compter les lignes cachées pour signaler un périmètre partiel** : fuite d'information. Déduction
  depuis les règles, post-MVP.
- **Pré-agrégation imposée** : préjugerait l'architecture. Non préjugée.
