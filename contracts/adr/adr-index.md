# Index des ADR — Fabrica

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
- **0003** (P) Contrat d'IHM piloté par le métamodèle (formulaires/listes).
- **0006** (P) Effets multi-canaux : catalogue, régimes d'application. *(remplace l'ancien 0006 IHM)*
- **0015** (P) Moteur de règles : composant générique encapsulé.
- **0016** (P) Moteur des effets : orchestration, point fixe, application.
- **0019** (P) Cycle de vie des objets : états et transitions.

## Transverse
- **0014** (P) Déploiement à chaud : instance, versions, rolling update.
- **0017** (P) Internationalisation & localisation : traduction, formats, préférences.

## Différés (identifiés, non traités v1)
- **0021** (D) Re-classification d'un objet (changement de sys_class).
- **0022** (D) Collaboration temps réel.
- **0024** (D) Chiffrement des données (« C » du cadre DICT).

## Numéros
- **0015** a d'abord porté « effets multi-canaux », déplacé vers **0006** ; réattribué au moteur de règles.
- Pas de trou ; **0029** est le prochain libre.

## Catalogues (contracts/catalogues/)
Registres des familles fermées, renvoyant aux ADR : **catalogue-capacites.md** (ADR-0025), **catalogue-effets.md** (ADR-0006), **catalogue-natures-acl.md** (ADR-0012), **catalogue-canaux.md** (ADR-0006). Motif : constitution § Catalogues.

## Dettes — ADR à écrire (référencés, non rédigés)
**Structurants / prioritaires :**
- **Points d'entrée / scripts** (confluent — à traiter en dernier).

**Autres :**
- Patron « gabarit fourni mais non appliqué, copié sur décision projet » (à formaliser au 2e usage).
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
