# ADR-0036 — Construire Fabrica plutôt qu'adopter un framework existant (make or buy)

**Statut :** Proposé — 2026-09-29
**Portée :** décision fondatrice sur la **nature du projet** (prolonge l'ADR-0001). Réévaluable en fin
de MVP.

---

## Contexte

Une fois les grands principes posés (ADR 0001 à 0035, tool-0001 à 0003), une analyse *make or buy* a
comparé Fabrica aux projets open source de même philosophie : un moteur générique piloté par
métadonnées, qui produit schéma de données, API, formulaires et listes.

**Critères retenus :**
- **fonctionnels** : facilité de paramétrage ; scripts appliqués sur tous les canaux ; droits fins
  (rôles, groupes, ACL, filtrage des lignes) ; entités et cycle de vie ; exposition (API) ;
  import/export ; reporting ; héritage ; connecteurs vers l'extérieur ; déploiement et propagation
  du paramétrage entre environnements ;
- **modèle économique** : **pur open source exigé** — pas d'open core (édition communautaire plus
  édition payante) ;
- **pérennité** : outil mûr, pérenne et bien soutenu, ou prometteur.

Les critères **techniques** (langage, base de données) ont été **délibérément exclus** : s'il adopte
un framework, le projet prend sa pile native.

Les constats ci-dessous reflètent l'état vérifié en **septembre 2026** ; licences et modèles
économiques évoluent (plusieurs ont changé récemment).

## Résultats par outil

**Frappe Framework — le plus adapté, non retenu pour l'instant.** Pur open source (licence MIT, aucune
fonctionnalité payante ; l'éditeur vit de l'hébergement). Couvre l'essentiel du besoin : DocTypes
paramétrables dans l'interface et écrits en JSON versionné, scripts serveur sur les événements de
document (tous canaux), workflows déclaratifs (états, transitions, rôles), droits par niveau de champ,
filtrage des lignes (restrictions par enregistrement lié, scripts de filtrage des requêtes), import,
rapports, traductions, personnalisation d'un DocType livré sans le modifier. Limites relevées : pas
d'héritage ; rôles non composables ; filtrage des lignes applicatif, dont la couverture sur tous les
canaux reste à vérifier ; propagation du paramétrage stocké en base par des exports à déclarer
(« fixtures »), avec un risque de divergence silencieuse entre environnements. **Frappe reste
l'alternative de référence** pour le référentiel EA.

**Écartés par le critère économique (open core) :**
- **Odoo Community** : l'édition Enterprise, propriétaire, porte l'outil de paramétrage visuel, les
  tableaux de bord avancés et la plateforme officielle de migration. Framework « code d'abord » ;
  cycle de vie codé.
- **NocoBase** : noyau sous Apache-2.0 depuis février 2026, mais éditions commerciales portant
  notamment la gestion des migrations et des livraisons entre environnements, le cluster, l'audit et
  les circuits d'approbation. Configuration stockée en base.
- **Twenty** : code sous AGPL-3.0, mais droits par ligne, SSO et journaux d'audit réservés au plan
  payant. Orienté CRM ; ni cycle de vie ni héritage.
- **Axelor** : communauté sous AGPL, éditions Pro et Enterprise payantes ; écosystème étroit.

**Écartés pour d'autres raisons :**
- **Tryton** : pur open source (GPL-3.0, porté par une fondation), socle solide, mais tout se code
  (aucun paramétrage visuel) et son interface web est datée.
- **Directus** : licence Monospace Sustainable Core, non open source au sens OSI.
- **Payload** : MIT, mais CMS dont la configuration est du code compilé (pas de métamodèle à
  l'exécution, donc pas d'auto-application) ; racheté par Figma en 2025.
- **iTop** : produit de CMDB/ITSM (AGPL), pas un moteur générique. Leçon retenue : les changements de
  modèle de données sont très coûteux pour les clients existants.
- **Hors catégorie** : constructeurs d'écrans (Appsmith, ToolJet, Budibase), tableurs sur base
  (NocoDB, Baserow), couches d'API sur PostgreSQL (PostgREST, pg_graphql, Supabase).

## Décision

**Construire Fabrica.** Le motif principal n'est pas un manque de Frappe, mais l'**objectif du
projet** : Fabrica est d'abord le terrain d'une expérience de **développement assisté par IA avec
Spec Kit** sur un vrai produit (ADR-0001). La valeur attendue de cette expérience est jugée supérieure
au gain de temps qu'apporterait Frappe. Le **référentiel EA** reste le premier usage et le banc de
validation.

Trajectoire : mener le **MVP de Fabrica**, puis y appliquer un **MVP du référentiel EA** (à définir).
La suite sera décidée selon les difficultés rencontrées et le reste à faire pour rejoindre la
couverture fonctionnelle de Frappe.

## Pas de critère chiffré de réévaluation (choix assumé)

Aucun seuil n'est fixé à l'avance pour un retour vers Frappe : la réévaluation se fera **au jugement**,
en fin de MVP. Le risque identifié est **accepté en connaissance de cause** : après un investissement
important, le biais des coûts engagés peut rendre difficile la décision d'adopter Frappe, même si elle
devenait la meilleure option.

## Ce que Fabrica garde comme différenciateurs

Relevés pendant l'analyse, aucun des outils comparés ne les réunit :
- une **règle d'effet déclarée une fois**, projetée dans l'interface **et** imposée sur tous les
  canaux (ADR-0006) — ailleurs, règle d'interface et contrôle serveur s'écrivent séparément ;
- un **cycle de vie** qui garantit qu'un objet reste dans un état légal (ADR-0019) ;
- des **rôles composables** et un filtrage des lignes au plus près des données (ADR-0012, 0026) ;
- **tout le paramétrage écrit dans git par construction** (tool-0001, tool-0003), sans liste d'exports
  à tenir à jour, donc sans divergence silencieuse ;
- la **mesurabilité DICT** (constitution) et la **revue de montée de version** (tool-0002).

## Conséquences

- L'analyse **valide la conception** : Frappe met en œuvre depuis des années plusieurs choix que nous
  avions faits de notre côté (métamodèle en JSON versionné avec ordre des champs, personnalisation
  d'un modèle livré sans le modifier, numérotation préfixée, suivi des versions, workflows à états).
- Elle **éclaire les manques** de Fabrica, qui devront être comblés pour rejoindre Frappe : navigation
  (espaces de travail), import de données, recherche globale, pièces jointes, commentaires,
  assignations, notifications. Plusieurs sont déjà inscrits comme dettes (index, revue du 2026-09-26).
- Si le projet bascule un jour vers Frappe pour le référentiel EA, cet ADR sera remplacé par un nouvel
  ADR (constitution, « Cycle de vie des ADR »).
