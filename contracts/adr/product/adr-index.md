# Index des ADR — Fabrica / **Produit** (product)

> Famille **Produit** : comportement de Fabrica et de l'application générée. Ce sont des décisions
> de conception dont beaucoup sont des **bonnes pratiques réutilisables** (au-delà de Fabrica).
> La famille **Outillage** (fabrication/packaging/déploiement/exploitation) est dans `../tool/`.

> Vue de lecture, tenue à jour au fil de l'eau. **Les numéros sont des identifiants, pas un ordre de
> lecture** : cet index donne l'ordre logique et les liens. À vérifier contre le dépôt git (source de
> vérité). Statuts : **P** = Proposé, **D** = Différé.

## Fondations & méthode
- **Constitution** — 7 principes (session jetable, deux sources de vérité, déclaratif, dépendance
  cœur, rayon de destruction, volatilité, caler la frontière) + contrat public du cœur + **cadre DICT** (sécurité de l'information, mesurabilité).
- **0001** (P) Nature du projet : produit-socle réutilisable, banc de validation EA.
- **0002** (P) Extraction du cœur en paquet versionné, fenêtre de régénération.
- **0013** (P) Anatomie de Fabrica et ligne de propriété (Fabrica / projet / instance).

## Persistance & structure de données
- **0004** (P) PostgreSQL.
- **0005** (P) Matérialisation de l'héritage (table par classe, ligne par niveau).
- **0018** (P) Listes de valeurs (table de référence, désactivation, conditionnement).
- **0020** (P) Colonnes système (sys_id, dates, _by, sys_version, sys_class).
- **0023** (P) Relations entre entités (N:1/1:N bidirectionnel, N:N en entité de liaison, `actif`).
- **0025** (P) Capacités d'entité (mécanisme ; catalogue `contracts/catalogues/catalogue-capacites.md`).
- **0027** (P) Historisation des changements (capacité `historiser`).
- **0028** (P) Audit de traçabilité (« T » de DICT ; capacité système `audit`).

## Moteur GraphQL & isolation
- **0007** (P) Isolation des projets vis-à-vis du moteur.
- **0011** (P) Moteur GraphQL : schéma généré par Fabrica, exécution par Grafast.

## Sécurité, identité, autorisation
- **0008** (P) L'API est la frontière de sécurité ; friction raisonnable contre le shadow IT (BFF différé).
- **0009** (D) Gouvernance de l'extraction / shadow IT.
- **0010** (P) Identité : frontière unique, propagation SET LOCAL, auth locale de secours.
- **0012** (P) Modèle d'autorisation : QUI-utilisateur, quadruplet ACL, rôles, groupes, administration.
- **0026** (P) Autorisation par ligne (RLS) : axes de gouvernance, accroche, conditions.

## Règles, effets, comportements
- **0003** (P) Contrat d'IHM piloté par le métamodèle (formulaires/listes). ↔ tool: éditeur de vues (dette)
- **0030** (P) Rendu de l'IHM : interprète dynamique React/TS, catalogue de composants. ↔ tool: éditeur de vues (dette)
- **0031** (P) Mécanisme de script : contexte (objet courant), sandbox pur, contrat. ↔ tool-0003 (mise à disposition du code)
- **0032** (P) Attributs calculés & valeur d'affichage (display value) — 1er client de 0031.
- **0033** (P) Numérotation : capacité `numérotée`, identifiant fonctionnel `number` (6 chiffres).
- **0034** (P) Forme des URLs : `/{code_entité}/{sys_id}` au MVP (numéro réservé).
- **0035** (P) Reporting v1 : rapports dans le métamodèle, un regroupement, comptage/somme/moyenne, tableaux de bord. ↔ tool: éditeur de rapports (dette)
- **0006** (P) Effets multi-canaux : catalogue, régimes d'application. *(remplace l'ancien 0006 IHM)*
- **0015** (P) Moteur de règles : composant générique encapsulé.
- **0016** (P) Moteur des effets : orchestration, point fixe, application.
- **0019** (P) Cycle de vie des objets : états et transitions.

## Transverse
- **0014** (P) Déploiement à chaud : instance, versions, rolling update.
- **0029** (P) Journalisation technique (logs) — observabilité exploitants/maintenance.
- **0017** (P) Internationalisation & localisation : traduction, formats, préférences.

## Différés (identifiés, non traités v1)
- **0021** (D) Re-classification d'un objet (changement de sys_class).
- **0022** (D) Collaboration temps réel.
- **0024** (D) Chiffrement des données (« C » du cadre DICT).

## Numéros
- **0015** a d'abord porté « effets multi-canaux », déplacé vers **0006** ; réattribué au moteur de règles.
- Pas de trou ; **0036** est le prochain libre.

## Catalogues (contracts/catalogues/)
Registres des familles fermées, renvoyant aux ADR : **catalogue-capacites.md** (ADR-0025), **catalogue-effets.md** (ADR-0006), **catalogue-natures-acl.md** (ADR-0012), **catalogue-canaux.md** (ADR-0006). Motif : constitution § Catalogues.

## Notes de conception (propriétés, pas des ADR)
- **Ordre des attributs** : propriété du **métamodèle** (l'ordre dans la liste des attributs d'une
  entité). Assure la **stabilité/canonicité** des fichiers JSON (tool-0001) et sert de **défaut** aux
  usages (formulaire, liste, API). Surcharge par vue : réservée. → à porter dans l'ADR métamodèle /
  le méta-métamodèle.

## Revue du 2026-09-26 — trous et vices cachés identifiés

**À traiter pour le MVP (avant la première génération) :**
- **Contrat de déclaration du métamodèle** (le trou central, annoncé par l'ADR-0003) : propriétés
  d'une entité et d'un attribut, **types d'attributs**, **contraintes** (longueur, bornes, précision
  décimale, **unicité** — y compris face à la suppression logique `actif`), **valeurs par défaut**,
  héritage, configuration des capacités. Débouche sur le méta-métamodèle v0.
- **Langage de conditions déclaratives** commun (effets, conditionnement de listes, filtres de
  rapports, RLS) : opérateurs, ET/OU, référence à l'utilisateur et à l'état. Et son emplacement dans
  l'arborescence (aucun fichier de règles prévu par tool-0003).
- **Évolution du schéma projet** quand le métamodèle change (ajout, retrait, changement de type sur
  une table peuplée) + **recalcul en masse** des attributs calculés matérialisés quand leur script
  change.
- **Navigation** : menus, page d'accueil, regroupement des entités en domaines ; comportement des
  listes (pagination, tri, filtres, recherche).
- **Droits et affichage indirect** : l'historique (ADR-0027) et le display value d'une référence
  (ADR-0030/0032) ne doivent pas montrer ce que le lecteur n'a pas le droit de lire (axe structurel
  et RLS). Objets `actif=faux` dans l'autocomplétion d'un champ Référence.
- **Sécurité de base** : nettoyage du texte riche côté serveur (XSS stocké), limites de temps et de
  mémoire des scripts (ADR-0031), secrets injectés à l'exécution (jamais dans l'image ni dans git),
  pas de mot de passe administrateur par défaut.
- **Authentification concrète du MVP** (ADR-0010) : gestion des comptes, hachage des mots de passe,
  expiration de session.
- **Fuseau horaire** des dates/heures (stockage UTC, affichage selon l'utilisateur) — absent de
  l'ADR-0017. **Recherche insensible aux accents** pour l'autocomplétion.

**Post-MVP, avec précaution immédiate :**
- **Données personnelles (RGPD)** : effacement face à la suppression logique et à l'historique en
  ajout seul. Précaution dès maintenant : ne jamais recopier de nom dans l'historique, l'audit ou
  les logs (stocker des `sys_id`, résoudre à la lecture — déjà le choix de l'ADR-0027).
- **Accessibilité** (RGAA, WCAG) : critère de choix des bibliothèques de composants dès la phase
  technique.
- **Exemples de domaine** dans les ADR (`serveur`, `ticket`, `u_task`…) : les marquer comme
  illustrations, pour qu'ils ne fuient pas dans le code du cœur (Principe IV).

## Dettes — ADR à écrire (référencés, non rédigés)
**Structurants / prioritaires :**
- **Points d'entrée / scripts** (confluent — à traiter en dernier).

**Autres :**
- Extension des entités Fabrica par le projet (attributs ajoutés aux tables système, avec vues/formulaires/effets associés).
- Reporting post-v1 : en-tête des conditions d'exécution (impératif, sans fuite), rapports utilisateur, multi-niveaux, cross-relation, programmés/exportés, protections de performance.
- Dettes de migration / gate de packaging (décisions projet exigées à la montée de version Fabrica).
- Comment un projet définit son usage de Fabrica dans toutes ses phases (dev, run, packaging, déploiement, sauvegarde, clonage) — outillage.
- Attributs calculés / dérivés (dont display value « prénom nom »).
- Observabilité de la sécurité / conformité (mesurer le niveau DICT effectivement atteint).
- Règles d'intégrité (frontière intégrité/autorisation).
- Historisation / traçabilité (mécanisme uniforme tout objet).
- Journalisation / observabilité (logs techniques, niveaux façon log4j).
- Numérotation (capacité « numérotée » : number, préfixe, séquence).
- Capacité d'entité « auditée », « affectable »… (au fil des besoins).
- Affectation / responsabilité (version ultérieure).
- Liens inter-entités (cascade inter-objets).
- Sources de données / ingestion.
- Projection des listes dans le contrat d'API (versionnement).
- Surcharge des traductions (extension additive).
- Métamodèle réflexif en table (principe général).
- Saisie / source de vérité du métamodèle (édition tabulaire vs git).
- Fonctions d'administration / introspection (sys_id, YAML/GraphQL d'un objet…).
- Forme des URLs (lisibilité/partage vs révélation de structure).
