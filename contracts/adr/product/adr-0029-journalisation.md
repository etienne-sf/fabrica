# ADR-0029 — Journalisation technique (logs)

**Statut :** Proposé — 2026-07-11
**Portée :** décision du **cœur** (Fabrica), générique (Principe IV). Troisième membre de la famille
traçabilité, distinct de l'historisation (ADR-0027, utilisateurs) et de l'audit (ADR-0028, sécurité).
À éclater en `contracts/adr/0029-journalisation.md`.

---

## Contexte

La **journalisation** (logs) sert l'**observabilité** : comprendre et analyser le **comportement de
la plateforme**. Public : **exploitants** et **équipe de maintenance/projet** (MCO), y compris pour
des **analyses fonctionnelles**. Un log est généralement associé à un **événement d'exécution** —
sans que ce soit une contrainte absolue.

Frontière de la famille :
- **Historisation** (ADR-0027) : pour les **utilisateurs** — quelles modifications sur un objet.
- **Audit** (ADR-0028) : pour la **sécurité** — le « T » de DICT.
- **Journalisation** (ici) : pour les **exploitants/maintenance** — le comportement de la plateforme.

## Décision — journalisation native, non scriptée

- Fabrica **journalise elle-même** aux points où le comportement peut différer de l'attendu : règles,
  effets, surveillance des appels d'API, erreurs JS navigateur, crash serveur, etc. Le projet
  **n'a pas à scripter** pour obtenir ces logs (cohérent « minimiser le bespoke »).
- Le **code projet peut** journaliser via une **API de log** exposée par Fabrica (une capacité de
  l'API Fabrica, pas un retour de script — cf. ADR-0017).
- **Journaliser le comportement complet** (succès **et** échecs), pas seulement les erreurs :
  « 100 erreurs/jour » n'a de sens que rapporté au volume total (le **nominal** donne le
  dénominateur nécessaire à l'analyse).

## Décision — niveaux et configuration

- **Niveaux techniques** (debug/info/warn/error…), **seuil réglable**, **affinable par composant**
  (façon log4j). **Non traduits** (destinés aux exploitants/devs), distincts du *niveau de message
  utilisateur* (ADR-0017).
- **Bibliothèque de journalisation standard** : Fabrica orchestre (points de log, format), la
  bibliothèque enregistre. Choix **différé à l'implémentation** (comme le moteur de règles, l'i18n).
- **Configuration** : gouvernée par le motif **gabarit fourni** (constitution) — Fabrica livre une
  config de log par défaut (minimale mais réelle dès le départ), le projet la personnalise ; les
  ajouts de points de log par une nouvelle version de Fabrica sont additifs (enrichissement), les
  modifications d'éléments personnalisés sont proposées. *(Le cas d'un **nouveau crochet de log
  transverse** exigeant une décision du projet à la montée de version relève de l'ADR « dettes de
  migration / gate de packaging » — hors périmètre ici.)*

## Décision — stockage

- **En base**, en **tables** (pas en fichier) : c'est ce qui rend les logs **consultables et
  interrogeables** en SQL (un fichier n'est pas interrogeable — c'est déjà bien mieux qu'un fichier
  accessible aux seuls exploitants).
- **Pas de schéma séparé au démarrage** : la valeur (isolation) ne justifie pas la complexité
  d'exploitation induite (sauvegarde, clonage d'environnement). Réservé si le volume l'exige un jour.
- **Écriture asynchrone / bufferisée** : le log **ne ralentit pas** l'opération métier (il n'est pas
  dans la transaction critique).
- **Purgeable / rétention** : les logs ne sont **pas une preuve** (contrairement à l'audit) — ils se
  **purgent** selon une politique de rétention, pour maîtriser le volume.

## Décision — consultation, pas analyse lourde

- **Consultation ponctuelle** assurée : rechercher/filtrer, voir les logs d'une requête, lister les
  erreurs récentes — requêtes légères, filtrées.
- **L'analyse lourde** (tableaux de bord, agrégations, tendances) **relève du reporting** (ADR
  distinct) et **ne doit pas écrouler la plateforme** (cf. ServiceNow, qui interdit par défaut les
  dashboards sur les logs). Le **comment** (index, moteur analytique, export, éventuelle
  pré-agrégation…) est **laissé au reporting** — le présent ADR **ne préjuge pas** de son
  architecture.

## Hors périmètre (renvois / dettes)

- **Reporting** (tableaux de bord, analyses, alertes) : **gros sujet distinct**, à traiter dans un
  second temps (ServiceNow en a fait un module séparé). Vaut pour les logs comme pour toute donnée.
- **Collecte / externalisation** (SIEM, puits de logs type ELK/Loki) : besoin futur identifié,
  réservé.
- **Dettes de migration / gate de packaging** (crochets transverses exigeant une décision projet à
  la montée de version ; tableau de bord de dettes plutôt qu'un wizard ; blocage du passage en RUN
  tant que non traité) : **ADR distinct**.
- **Deux niveaux fonctionnel/infra** : écartés (ne pas cloisonner les logs par nature — cela
  empêcherait la corrélation, ex. un problème fonctionnel dû à un problème infra).

## Alternatives écartées

- **Logs en fichier** : non interrogeables ; analyse impossible sans outil externe. Table en base.
- **Logs synchrones dans la base métier** : *ça*, c'est l'anti-pattern (ralentit le métier,
  concurrence les I/O). Évité par l'asynchrone (+ purge). En base ≠ anti-pattern ; *synchrone
  haute-fréquence dans la base métier* = anti-pattern.
- **Schéma séparé au démarrage** : complexité d'exploitation non justifiée. Réservé.
- **Ne logger que les erreurs** : prive du dénominateur. On logue succès et échecs.
- **Analyse lourde en direct sur les logs bruts** : écroule la plateforme. Renvoyée au reporting.
