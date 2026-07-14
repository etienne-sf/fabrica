# ADR-0008 — L'API est la frontière de sécurité ; le front-end n'en fournit aucune

**Statut :** Proposé — 2026-07-11
**Portée :** décision du **cœur** (Fabrica), générique (Principe IV). Principe-parent de
l'ADR-0009 (gouvernance de l'extraction). À éclater en
`contracts/adr/0008-api-frontiere-securite.md`.

---

## Contexte

Le front-end de Fabrica (déployé au navigateur) appelle le back-end en GraphQL. Question
naturelle : comment s'assurer que les appels à l'API sont légitimes, c'est-à-dire qu'ils
proviennent bien du front-end officiel, et empêcher un utilisateur (ou un service) de
développer une application « dans son coin » qui appellerait l'API avec ses propres droits ?

## Décision — le principe (la garantie)

**La sécurité est portée par le serveur et la base ; l'API est la frontière de sécurité. Le
front-end ne fournit AUCUNE sécurité — il est du confort d'affichage.**

- Toute opération est autorisée **au serveur** (autorisation d'écriture par (entité,
  attribut) ; à terme, RLS en lecture) et contrainte **à la base** (invariants, FK,
  contraintes), **quel que soit l'appelant**. Un appel issu d'un script maison n'obtient que
  les droits de son jeton, exactement comme le front-end officiel. Le serveur ne fait **jamais**
  confiance à un contrôle amont (cf. constitution, « serveur autoritaire, client projeté »).
- **Il est impossible d'authentifier que l'appel provient du front-end officiel.** Le
  navigateur est sous le contrôle total de l'utilisateur : toute « preuve » d'origine que le
  front-end émet (en-tête, clé, jeton) est extractible et rejouable. Une requête du SPA
  légitime et une requête d'un script qui l'imite sont, au bit près, indiscernables. La
  sécurité ne DOIT donc **jamais** dépendre de l'identification du *client*.
- **Réserve technologique.** Cette impossibilité est constatée dans l'état de l'art actuel. Si
  une technologie future permettait une attestation de client réellement fiable, la question
  sera réexaminée à ce moment — elle n'est pas comptée sur aujourd'hui.

## Décision — rendre l'usage illégitime difficile (friction, PAS sécurité)

**En complément — jamais en remplacement — de la garantie serveur, Fabrica élève la barre**
contre le développement d'un client « shadow » : le ralentir, le rendre coûteux, visible et
attribuable. Ces mesures sont de la **friction et de la dissuasion, pas des garanties**.

> **Test de non-régression conceptuel :** si l'on retirait *toutes* ces mesures de friction,
> ce qu'un appelant autorisé peut faire ne DOIT PAS changer — seule l'autorisation serveur le
> détermine. Aucune de ces mesures n'est comptée comme sécurité. Les confondre recrée
> exactement l'illusion que cet ADR combat.

**Principe directeur : route pavée d'un côté, friction de l'autre.** On ne durcit pas l'API
en général (cela pénaliserait les applications externes *légitimes* qu'on doit permettre). On
rend **facile** l'accès programmatique *sanctionné* et **coûteux** le détournement du canal du
front-end. La friction tombe sur le chemin non sanctionné ; le chemin sanctionné est dégagé.

Pistes, classées par valeur réelle :

**Forte valeur, faible coût (à privilégier) :**
- **Minimisation de la surface d'API** : n'exposer que ce dont le front-end a besoin — pas de
  surface de requête générique, pas d'export de masse ni de filtrage arbitraire par défaut.
  Moins il y a à exploiter, plus un client shadow est limité. (Cohérent avec l'ADR-0007 : la
  forme de l'API est le contrat de Fabrica, pas la sortie brute du moteur.)
- **Accès programmatique sanctionné et scopé** (comptes de service) : le besoin légitime ne
  passe plus par le vol du jeton du SPA ; le détournement devient l'anomalie qui se voit.
  *Gouvernance opérationnelle détaillée en ADR-0009.*
- **Quotas / limitation de débit** : étranglent l'aspiration de masse sans gêner l'usage
  humain. *Détaillé en ADR-0009.*
- **Audit d'accès** : rend tout usage attribuable ; le shadow IT prospère dans l'invisibilité.
  *Détaillé en ADR-0009.*

**Valeur moyenne, coût réel (option à peser) :**
- **Jetons de courte durée + rafraîchissement** : un jeton extrait du navigateur expire vite ;
  un client shadow doit réimplémenter tout le cycle d'authentification. Bonne hygiène par
  ailleurs. Ne prévient pas (automatisable), augmente l'effort.
- **Requêtes pré-enregistrées / liste blanche** (persisted queries) : le serveur n'accepte que
  les requêtes qu'il connaît du client officiel, refusant les requêtes ad hoc. Friction forte.
  **Coût réel** : rigidité, outillage ; et contournable (extraction des identifiants de
  requêtes). À réserver aux projets sensibles.
- **Détection d'anomalies / de volumes déviants** : élève le risque de détection.
  *Moteur reporté, cf. ADR-0009.*

**Faible valeur (à ne pas sur-investir — nommées pour éviter le piège) :**
- **Obfuscation du code front-end** : quasi sans valeur (dissuade le plus paresseux, rien de
  plus). Ne pas y consacrer d'effort.
- **Attestation de client sur le web ouvert / vérification d'origine forcée** : faible et
  controversée aujourd'hui — relève de la réserve technologique ci-dessus, pas d'une solution
  actuelle.

## Alternatives écartées (fausses sécurités)

Toutes reposent sur l'idée qu'on peut faire confiance à ce que l'utilisateur contrôle — donc
**pires qu'inutiles**, car elles créent un faux sentiment de sécurité :

- **Clé d'API dans le JS du front-end** : extractible immédiatement.
- **Vérification de l'en-tête `Origin` / `Referer`** : falsifiable hors navigateur (ces
  en-têtes ne sont imposés que *par* le navigateur, jamais garantis côté serveur).
- **CORS comme contrôle d'accès** : méprise fréquente. CORS protège les navigateurs *des
  utilisateurs* contre des requêtes cross-origin ; il n'empêche en rien un client non-navigateur
  (curl, script) d'appeler l'API. Ce n'est pas un contrôle d'accès serveur.
- **En-tête custom « secret » ajouté par le front** : falsifiable, même famille.

## Conséquences

- La sécurité réelle repose entièrement sur l'autorisation serveur/base — déjà décidée pour
  l'écriture (par attribut), réservée pour la lecture (RLS, inerte au démarrage).
- Les mesures de friction sont **optionnelles et configurables par criticité** (comme
  l'ADR-0009) : un projet EA en active peu, un projet CMDB davantage. Aucune n'est un
  fondement de sécurité.
- **À graver contre l'oubli :** ne jamais reclasser une mesure de friction en garantie de
  sécurité ; ne jamais retenter d'authentifier le client en croyant sécuriser quelque chose.
  Si quelqu'un (humain ou IA) propose « bloquons les appels hors du front-end », cet ADR est la
  réponse : c'est impossible, et la sécurité est ailleurs.