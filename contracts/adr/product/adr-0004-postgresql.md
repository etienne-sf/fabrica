## ADR-0004 — PostgreSQL comme moteur de base de données

**Statut :** Proposé — 2026-07-11

### Contexte
Le cœur doit porter ses invariants au plus près de la donnée (dernière ligne infranchissable), supporter des volumes potentiellement importants, rester open source et déployable en local (Docker) puis sur n'importe quel cloud. Un besoin futur de filtrage de lignes en lecture (RLS) et un besoin possible d'incorporer, à terme, un inventaire CMDB à très gros volume ont été identifiés.

### Décision
La base de données est **PostgreSQL**.

### Raison
- Open source ; encaisse les volumes (partitionnement, index) ; système de contraintes le plus riche (`DOMAIN`, `CHECK`, exclusion, `FOREIGN KEY`) pour porter les invariants du Principe d'intégrité ; **RLS** disponible pour un futur filtrage en lecture ; managé sur tous les clouds (portabilité acquise) ; meilleure cible de l'écosystème des moteurs GraphQL.

### Alternatives écartées
- *Base graphe (Neo4j)* : naturelle pour la traversée d'impact, mais faible précisément là où est l'exigence (contraintes, intégrité, RLS, gros volumes). La traversée se fait en SQL récursif à l'échelle du cœur. *Hedge* : extension **Apache AGE** (graphe sur Postgres) si la traversée devient centrale, sans quitter les garanties relationnelles.
- *Autres SGBD relationnels propriétaires* : écartés par l'exigence open source et portabilité.

### Conséquences
- Le cœur EA (milliers à dizaines de milliers d'objets gouvernés) et un éventuel firehose CMDB (100 M+ de lignes machine) devront être **deux sous-systèmes distincts** même sous un même moteur (design séparé, cf. ADR dédié [à écrire si/quand la CMDB est incorporée]).
- Les atouts spécifiques de Postgres (`DOMAIN`, `inet`, contraintes d'exclusion, RLS) sont supposés disponibles dans les décisions d'intégrité et d'autorisation à venir.
- Reste **ouvert et bloquant** : le choix du moteur GraphQL au-dessus de Postgres (Hasura vs PostGraphile vs serveur fait main), qui conditionne plusieurs ADR non encore rédigés.