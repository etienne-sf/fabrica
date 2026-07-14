# ADR-0010 — Identité : frontière unique et propagation jusqu'à la donnée

**Statut :** Proposé — 2026-07-11
**Portée :** décision du **cœur** (Fabrica), générique (Principe IV). Point d'ancrage
présupposé par l'ADR-0007 (isolation) et l'ADR-0008 (trajectoire BFF). À éclater en
`contracts/adr/0010-identite-propagation.md`.

> Cet ADR **possède** la plomberie d'identité et la frontière unique. Il **renvoie** sans les
> redécider : l'autorisation d'écriture par (entité, attribut) → constitution ; la RLS et
> l'indirection de lecture → réservées (Principe VII) ; le BFF → ADR-0008 ; la sécurité
> serveur → ADR-0008.

---

## Contexte

Toute autorisation étant portée par le serveur et la base (ADR-0008), il faut qu'une identité
**vérifiée** descende du point d'authentification jusqu'à la donnée, malgré un pool de
connexions mutualisé sous un utilisateur technique unique. Par ailleurs, la *provenance* de
cette identité est appelée à changer dans le temps (source provisoire → Keycloak → insertion
d'un BFF), sans que l'aval doive être réécrit à chaque fois.

## Décision — une frontière d'identité unique et substituable

**L'aval (la couche, la base) ne dépend que d'un fait : il reçoit une identité vérifiée. Il
ne dépend jamais de la *provenance* de cette identité.** Cette frontière unique absorbe,
successivement dans le temps, plusieurs implémentations sans réécriture de l'aval :

1. **Démarrage (le plus raisonnable)** : source provisoire — le back émet lui-même un **JWT
   signé** (clé locale) portant `user_id` + rôles/profils, vérifié en aval. Ni Keycloak, ni
   cookie, ni BFF, ni CSRF.
2. **Puis** : remplacement de l'**émetteur** par **Keycloak** (OIDC/OAuth, Authorization Code
   + PKCE ; vérification locale du JWT via JWKS, sans appel à Keycloak par requête).
   Substitution *indolore* : l'aval reçoit toujours un JWT vérifié, peu importe le signataire.
3. **Puis (différé)** : insertion du **BFF** (ADR-0008), qui change le *mode de transport* de
   l'identité entre le front et le serveur (jeton porté par le SPA → cookie de session vers le
   BFF). Côté serveur, la frontière reste stable ; côté navigateur, apparaît le besoin de
   protection CSRF (seule conséquence non indolore, cf. ADR-0008).

Le contrat de cette frontière : **« une identité vérifiée portant au minimum `user_id` et les
rôles/profils »**. Ce que le jeton transporte est un contrat à décider tôt (les *token
mappers* côté émetteur), car c'est lui qui relie « qui est connecté » à ce que la base pourra
lire.

## Décision — propagation jusqu'à la session PostgreSQL

Malgré le pool de connexions mutualisé, l'identité vérifiée descend jusqu'à la base par le
motif suivant :

- À chaque requête, sur la connexion empruntée au pool, la couche exécute dans la **même
  transaction** que l'opération : `SET LOCAL app.user_id = …` (+ rôles), déposant l'identité
  dans une variable de configuration de session.
- Les prédicats en base (une vue, une fonction, ou plus tard une politique RLS) lisent
  `current_setting('app.user_id')` — jamais `current_user` (qui est le rôle technique).
- L'utilisateur technique conserve ses droits *table* ; c'est le **contexte**, redéposé à
  chaque requête, qui porte l'identité réelle.

Propriétés clés :
- **`LOCAL` lie la variable à la transaction** → elle disparaît au `COMMIT`, donc **aucune
  fuite d'identité** vers la requête suivante d'un autre utilisateur sur la connexion recyclée.
- La valeur vient du **JWT vérifié**, non d'un paramètre client brut → non falsifiable.
- Cette plomberie est **indépendante de la politique** qu'elle sert : elle alimente aujourd'hui
  un prédicat de lecture `TRUE` (lecture ouverte à tous), demain une RLS fine — sans changer la
  tuyauterie (Principe VII : caler la frontière, pas la politique).

## Ce qui est calé maintenant vs réservé

**Calé dès le départ** (non reportable — rétro-ajout = réécriture de tous les accès) :
- la frontière d'identité unique (contrat « identité vérifiée ») ;
- la propagation `JWT vérifié → SET LOCAL → current_setting` ;
- l'**indirection de lecture** : jamais de tables nues ; lecture via vue/fonction dont le
  prédicat d'autorisation est aujourd'hui `TRUE`, prêt à recevoir une condition ;
- la **distinction des identités humain / compte de service** (prérequis de l'ADR-0009).

**Réservé, inerte au démarrage** (Principe VII) :
- la **RLS** en lecture (le prédicat reste `TRUE` tant que la visibilité différenciée n'est
  pas un besoin réel) ;
- Keycloak, le BFF (cf. ADR-0008), la détection de déviances (cf. ADR-0009).

## Invariant de sécurité (test gelé)

L'absence de fuite d'identité entre requêtes partageant une connexion recyclée est un
**invariant de sécurité**, pas un détail d'implémentation. Il DOIT figurer dans
`tests/acceptance/` : « deux requêtes successives d'utilisateurs différents sur la même
connexion ne voient jamais le contexte l'une de l'autre ».

## Points de vigilance

- **Pooler externe.** Un pooler agressif (PgBouncer en mode transaction/statement) peut casser
  l'hypothèse « même connexion, même transaction » et dissocier l'identité de la requête. À
  vérifier selon le montage ; sinon la propagation s'effondre silencieusement.
- **Compatibilité moteur.** Le `SET LOCAL` par transaction suppose que le moteur GraphQL ouvre
  une transaction par requête et laisse injecter ce préambule (Hasura via ses session
  variables ; PostGraphile via `pgSettings`). Élément à confirmer au choix du moteur (ADR à
  venir).

## Alternatives écartées

- **RLS basée sur les rôles PostgreSQL.** Inopérante avec un pool sous utilisateur technique
  unique (un seul rôle, tout-puissant). Remplacée par la RLS/prédicats pilotés par variable de
  session.
- **Faire dépendre l'aval de la provenance de l'identité** (câbler « SPA porteur de JWT » en
  dur). Rejetée : chaque changement de source (Keycloak, BFF) deviendrait une refonte. La
  frontière unique l'évite.
- **Transmettre l'identité applicative en paramètre client brut** (non issu d'un jeton
  vérifié). Falsifiable. Rejetée.

## Conséquences

- L'autorisation (par attribut en écriture ; RLS réservée en lecture) s'appuie sur cette
  identité de session ; sans cette plomberie, aucune autorisation proche de la donnée n'est
  possible.
- La substituabilité de la source d'identité (provisoire → Keycloak → BFF) est acquise *par
  construction*, tant que rien en aval ne présume la provenance.
- Le contrat « ce que le jeton transporte » est versionné comme le reste du contrat public
  (constitution, compatibilité ascendante) : l'ajouter des claims est additif, en retirer est
  cassant.