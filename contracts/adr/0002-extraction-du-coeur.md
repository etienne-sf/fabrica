## ADR-0002 — Extraction du cœur en paquet versionné et fenêtre de régénération

**Statut :** Proposé — 2026-07-11

### Contexte
Le cœur doit rester régénérable au début (pour bootstrapper via l'IA) tout en devenant
stable dès que des projets en dépendent. Ces deux exigences ne se contredisent que si l'on
ne date pas la fin de la fenêtre de régénération. Par ailleurs, plusieurs projets-clients
consommeront le cœur : la façon dont ils en dépendent doit être fixée dès maintenant, car
elle contraint la structure du dépôt.

### Décision
- **Fenêtre de régénération.** Tant qu'aucun projet-client réel ne dépend du cœur, celui-ci
  est un *spike* régénérable intégralement. À partir de la **première version publiable**
  consommée par un projet distinct, le cœur n'évolue plus que par amendement additif et
  versionné.
- **Bascule = événement observable.** La bascule est matérialisée par la **création d'un
  second dépôt** (le référentiel EA de production) qui *importe* le cœur versionné au lieu de
  le contenir.
- **Le cœur est conçu dès maintenant comme un paquet importable.** Dépendance à sens unique
  (client → cœur) ; aucune dépendance sortante du cœur vers un projet-client ; aucun nom de
  domaine dans le cœur (Principe IV). Le banc de validation intégré dépend du cœur, jamais
  l'inverse.
- **Cycle de vie de dépendance.** Chaque client épingle une version du cœur et adopte les
  montées à son rythme. Les manques du cœur découverts en développant un client sont corrigés
  dans le cœur, publiés en nouvelle version, puis remontés — **jamais par copie divergente**.

### Raison
- Dater la bascule rend cohérents le Principe VI (« la capacité à régénérer se construit dès
  la première ligne ») et l'usage réel (régénérer le cœur au tout début).
- Concevoir le cœur comme paquet dès le départ rend l'extraction du jour J **triviale** au
  lieu de douloureuse, et interdit la tentation de copier le code (source de dérive).

### Alternatives écartées
- *Séparer les dépôts plus tard sans préparation* : conduirait à copier le code du cœur
  plutôt qu'à l'importer, rouvrant la dérive entre deux copies. Écartée.
- *Bascule définie par une date ou un jalon subjectif* : non observable, sujette à
  interprétation. Remplacée par un critère matériel (naissance du second dépôt / première
  version publiable).

### Conséquences
- Le format de déclaration du métamodèle devient une **API publique versionnée** (cf.
  constitution, « Contrat public du cœur et compatibilité ascendante » ; motif expand/contract
  appliqué au contrat du cœur).
- La CI et le CODEOWNERS doivent, dès le dépôt du cœur, empêcher toute dépendance du cœur vers
  le banc de validation.
- Une définition de « prêt à publier en v1.0 » est requise (contrat de métamodèle stable sur
  quelques itérations, génération reproductible).
