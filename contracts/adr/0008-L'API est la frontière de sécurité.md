# ADR-0008 — L'API est la frontière de sécurité ; friction raisonnable contre le shadow IT

**Statut :** Proposé — 2026-07-11
**Portée :** décision du **cœur** (Fabrica), générique (Principe IV). Principe-parent de
l'ADR-0009 (gouvernance de l'extraction). À éclater en
`contracts/adr/0008-api-frontiere-securite.md`.

---

## Contexte

Le front-end de Fabrica (au navigateur) appelle le back-end en GraphQL. Question : comment
s'assurer que les appels sont légitimes, et empêcher un utilisateur (ou un service) de
scripter des appels « dans son coin » avec son compte standard, pour bâtir un usage — voire un
référentiel — parallèle ?

## Décision 1 — la garantie : sécurité serveur, jamais client

**La sécurité est portée par le serveur et la base ; l'API est la frontière de sécurité. Le
front-end ne fournit AUCUNE garantie de sécurité — il est du confort d'affichage.**

- Toute opération est autorisée **au serveur** (autorisation par (entité, attribut) ; à terme
  RLS en lecture) et contrainte **à la base**, **quel que soit l'appelant**. Un script maison
  n'obtient que les droits de son jeton, comme le front officiel. Le serveur ne fait jamais
  confiance à un contrôle amont (constitution : « serveur autoritaire, client projeté »).
- **Il est impossible d'authentifier que l'appel provient du front officiel.** Le navigateur
  est sous contrôle total de l'utilisateur : toute preuve d'origine émise par le front est
  extractible et rejouable ; une requête du SPA et celle d'un script qui l'imite sont
  indiscernables. La sécurité ne DOIT jamais dépendre de l'identification du client.
- **Réserve technologique.** Constat valable dans l'état de l'art actuel. Si une technologie
  future permettait une attestation de client réellement fiable, la question sera réexaminée
  alors ; elle n'est pas comptée sur aujourd'hui.

## Décision 2 — la friction : élever la barre, dans la mesure du raisonnable

**Il FAUT mettre en place (ou prévoir) tout ce qui, raisonnablement, freine ou complique le
scripting shadow d'un utilisateur muni de son compte standard.** Ces mesures ne sont **pas**
des garanties : elles ralentissent, renchérissent, rendent visible et attribuable, et
réduisent la surface exploitable. Elles s'ajoutent à la garantie serveur, jamais à sa place.

> **Test de non-régression conceptuel :** retirer *toutes* les mesures de friction ne DOIT
> pas changer ce qu'un appelant autorisé peut faire — seule l'autorisation serveur le
> détermine. Aucune friction n'est comptée comme sécurité ; les confondre recrée l'illusion
> que cet ADR combat.

**Principe directeur : route pavée d'un côté, friction de l'autre.** On ne durcit pas l'API en
général (cela pénaliserait les applications externes légitimes). On rend **facile** l'accès
programmatique *sanctionné* (comptes de service scopés) et **coûteux** le détournement du
canal du front. La friction tombe sur le chemin non sanctionné ; le chemin sanctionné est
dégagé.

Mesures à mettre en place / prévoir, classées par valeur réelle :

**Forte valeur — à mettre en place :**
- **BFF (Backend-for-Frontend) — DIFFÉRÉ, emplacement réservé** (voir section dédiée
  ci-dessous). Mesure forte pour le canal front (sécurité anti-XSS réelle + friction
  anti-shadow + réduction de surface), non construite au démarrage.
- **Minimisation de la surface d'API** : n'exposer que ce qui est nécessaire ; pas de surface
  générique, d'export de masse ni de filtrage arbitraire par défaut (cohérent ADR-0007 : la
  forme de l'API est le contrat de Fabrica). Le BFF renforce ce point pour le canal front.
- **Accès programmatique sanctionné et scopé** (comptes de service) : le besoin légitime ne
  passe plus par le vol du jeton du SPA ; le détournement devient l'anomalie visible.
  *Gouvernance détaillée en ADR-0009.*
- **Quotas / limitation de débit** et **audit d'accès** : étranglent l'aspiration de masse,
  rendent tout usage attribuable. *Détaillés en ADR-0009.*

**Valeur moyenne — à peser / prévoir :**
- **Jetons de courte durée + rafraîchissement** : un jeton (ou cookie) volé expire vite ;
  bonne hygiène par ailleurs. Augmente l'effort du client shadow sans le prévenir.
- **Requêtes pré-enregistrées / liste blanche** (persisted queries) : le serveur n'accepte que
  les requêtes connues du client officiel. Friction forte, mais **coût réel** (rigidité,
  outillage) et contournable. À réserver aux projets sensibles.
- **Détection d'anomalies / de volumes déviants** : élève le risque de détection. *Moteur
  reporté, cf. ADR-0009.*

**Faible valeur — à ne PAS sur-investir (nommées pour éviter le piège) :**
- **Vérification d'en-tête `Origin` / `Referer`** : quasi nulle en friction (falsifiable hors
  navigateur), et gêne les intégrations légitimes. À n'inclure que si réellement gratuit.
- **Obfuscation du code front** : quasi sans valeur. Pas d'effort à y consacrer.

## À NE PAS confondre avec de la sécurité (le seul vrai « écarté »)

Ce qui est écarté n'est pas la friction, c'est **l'idée qu'une de ces mesures constitue une
sécurité** :
- **Clé d'API dans le JS du front / en-tête secret ajouté par le front** : extractibles
  immédiatement — ni sécurité, ni même friction réelle. Pur théâtre, à ne pas mettre en place.
- **CORS n'est PAS un contrôle d'accès serveur** : il protège les navigateurs *des
  utilisateurs* contre des requêtes cross-origin ; il n'empêche en rien un client non-navigateur
  (curl, script) d'appeler l'API. **CORS reste nécessaire pour sa vraie fonction** (protection
  du navigateur), mais ne compte jamais comme anti-shadow.

Règle générale : toute mesure reposant sur la confiance en ce que l'utilisateur contrôle
(navigateur, jeton, en-têtes) peut servir de friction, **jamais** de garantie.

## BFF — différé, mais frontière d'identité calée dès le départ

Le BFF (le SPA ne détient plus de jeton ; il dialogue par cookie de session HttpOnly/SameSite,
le BFF gardant le jeton côté serveur et le relayant en aval) est **différé** : aucun composant
à écrire au démarrage. Ce qui compte est de le différer **sans se coûter cher plus tard**.

**Une seule frontière d'identité, substituable.** C'est la même frontière que celle posée pour
la source d'identité provisoire → Keycloak : l'aval (la couche) reçoit une **identité
vérifiée**, sans savoir d'où elle vient. Cette frontière unique absorbe successivement, dans le
temps, plusieurs implémentations sans réécriture de l'aval :
1. **Démarrage (le plus raisonnable)** : source provisoire — le back émet un JWT signé,
   vérifié, `SET LOCAL` dans la transaction. Ni Keycloak, ni cookie, ni BFF, ni CSRF.
2. **Puis** : remplacement de l'**émetteur** par Keycloak (OIDC). Substitution *indolore* :
   l'aval reçoit toujours un JWT vérifié, peu importe le signataire.
3. **Puis (différé)** : insertion du **BFF**, qui change le *mode de transport* de l'identité
   entre le front et le serveur (jeton porté par le SPA → cookie de session vers le BFF).

**Ce qui se cale maintenant** (pour que le BFF reste un ajout, pas une refonte) : le front et
la couche ne présument PAS un modèle « SPA porteur de jeton » gravé en dur. Le point où le
front obtient et présente son identité est une frontière propre, indépendante de la *provenance*
de l'identité.

**Ce qui se diffère entièrement** : le composant BFF lui-même (garde du jeton côté serveur,
gestion de session cookie, surface étroite).

**La seule conséquence NON indolore, à assumer en connaissance de cause :** le passage du
modèle à jeton (en-tête) au modèle à cookie introduit le besoin d'une **protection CSRF**
explicite, absente du modèle à jeton. La frontière d'identité absorbe le reste ; le CSRF, lui,
est un ajout réel au jour du BFF. Côté *serveur*, la frontière reste stable (la couche reçoit
une identité vérifiée) ; côté *navigateur*, le mode de session change — c'est là, et là
seulement, que « j'assumerai le changement » a un contenu concret.

## Conséquences

- La sécurité réelle repose entièrement sur l'autorisation serveur/base — décidée pour
  l'écriture (par attribut), réservée pour la lecture (RLS, inerte au démarrage).
- La friction est **à mettre en place dans la mesure du raisonnable** et **configurable par
  criticité** (comme ADR-0009) : un projet EA en active moins, un projet CMDB davantage.
- Le **BFF est différé, emplacement réservé** (cf. section dédiée). Retenu comme cible pour
  le canal front, non construit au démarrage. Son insertion future est rendue indolore par la
  frontière d'identité unique et substituable ; seule exception à l'indolence : l'arrivée de la
  protection CSRF au passage cookie. À cadrer toujours comme friction + garde de jeton, jamais
  « anti-shadow garanti ».
- **À graver contre l'oubli :** ne jamais reclasser une friction en garantie ; ne jamais
  retenter d'authentifier le client en croyant sécuriser. Si quelqu'un propose « bloquons les
  appels hors du front », la réponse est ici : c'est impossible, la sécurité est ailleurs, et
  ce qu'on fait à la place, c'est de la friction raisonnable + une route sanctionnée.