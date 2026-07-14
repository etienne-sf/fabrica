## ADR-0001 — Nature du projet : produit-socle réutilisable, validé sur métamodèle EA réel

**Statut :** Proposé — 2026-07-11

### Contexte
Le projet a démarré comme banc d'essai du développement assisté par IA, en prenant un référentiel d'architecture d'entreprise (EA) comme cas d'usage bien maîtrisé. Au fil de la conception, l'objectif a évolué : le cœur (métamodèle + outils de génération) est destiné à être réutilisé par plusieurs projets successifs, le référentiel EA n'étant que le premier. Cette évolution change le niveau d'exigence et devait être actée explicitement, car elle métait restée ambiguë (« banc d'essai vs produit »).

### Décision
Fabrica est un **produit-socle réutilisable** : un moteur générique piloté par métamodèle,
destiné à servir plusieurs projets-clients. Le référentiel EA en est le **premier client**
et sert de **banc de validation et de vitrine**, sans définir la nature de Fabrica.

Le banc de test intégré au dépôt du cœur est fondé sur le **métamodèle EA réel**, et non sur
un métamodèle-jouet dédié conçu pour stresser la généricité.

### Raison
- Le niveau d'exigence est celui d'un composant dont d'autres projets dépendront (stabilité,
  compatibilité ascendante, documentation du contrat), pas celui d'un livrable unique.
- Utiliser l'EA réel comme banc évite de maintenir un second fonctionnel sans valeur propre,
  et sert simultanément de démonstration client — double usage assumé.

### Risque explicitement assumé
La généricité du cœur **n'est pas prouvée** par un second métamodèle dissemblable. Une
hypothèse spécifique à l'EA pourrait remonter dans le cœur sans être détectée par le seul
banc EA. Ce risque est accepté au regard du coût d'un second fonctionnel. **La garde qui
subsiste est le Principe IV** de la constitution (le cœur ne nomme aucun concept de
domaine) ; sa vérification reste **humaine**, à défaut de canari automatique.

### Alternatives écartées
- *Banc de validation fondé sur un ou plusieurs métamodèles-jouets dissemblables* : prouverait
  la généricité et testerait la compatibilité ascendante du cœur sur des formes variées, mais
  impose de concevoir et maintenir un fonctionnel artificiel sans valeur de démonstration.
  Écartée pour non-rentabilité au stade actuel ; réévaluable si un deuxième client réel arrive.
- *Projet unique (référentiel EA seul, sans cœur réutilisable)* : plus simple, mais renonce à
  la valeur d'accélération et de réemploi qui est l'objectif.

### Conséquences
- La bascule « spike régénérable → cœur stable » est un événement observable (cf. ADR-0002).
- La vérification manuelle du Principe IV devient un point de vigilance permanent en revue.
- Un canari automatique de généricité reste une option future peu coûteuse si le besoin de
  garantie monte (ex. arrivée d'un deuxième client).