ADR — Contrat d'IHM piloté par le métamodèle (formulaires et listes)
Décision. Les formulaires et listes sont dérivés du métamodèle, sur trois couches distinctes :

Contrat (métamodèle, versionné) : structure du formulaire, colonnes par défaut de la liste, bornes de personnalisation. Ne re-déclare aucun attribut — sélectionne et ordonne des attributs déjà déclarés.
Préférence (donnée utilisateur, en base, par utilisateur) : colonnes choisies et leur ordre.
Rendu (périphérie, régénérable) : lit contrat + préférence, affiche.

Déclarations.

label et listable portés par l'attribut (source unique, consommée par formulaire et liste). L'ensemble des colonnes offertes se dérive des attributs listable: true — pas de liste colonnes_disponibles re-déclarée.
La vue liste déclare colonnes_defaut, et une personnalisation bornée : colonnes_verrouillees (jamais retirables), max_colonnes (borne de perf), ajout_retrait, reordonnancement.
Contrainte de cohérence imposée par la validation du métamodèle : toute colonne par défaut ou verrouillée doit être un attribut listable: true.

Sémantique de composition : option A stricte.

Préférence absolue : la préférence est la liste complète et ordonnée voulue (pas un différentiel).
Pas de propagation des nouveaux défauts : un utilisateur ayant une préférence enregistrée reste sur sa vue ; une colonne ajoutée aux colonnes_defaut en v2 n'apparaît pas chez lui.
Filtrage systématique contre le métamodèle courant à l'affichage (non optionnel) : à la lecture, la préférence est intersectée avec les attributs actuellement listable. Une colonne référençant un attribut disparu ou passé listable: false est ignorée silencieusement. Ne surpasse jamais colonnes_verrouillees ni max_colonnes. Le métamodèle est autoritaire, la préférence subordonnée.

Autorisation, deux axes distincts.

Édition de la donnée : réutilise sans exception l'autorisation ecriture: par (entité, attribut) du §11. Formulaire et cellule de liste sont deux vues du même contrat de droits. Le rendu interroge l'éditabilité par cellule (ligne × attribut × utilisateur), sans mettre en cache « colonne éditable » — pour absorber sans refonte les règles conditionnelles futures.
Édition de la préférence : un utilisateur n'écrit que sa propre préférence (user_id = utilisateur authentifié). Permission distincte de la précédente, sur un objet distinct.

Frontière d'écriture. L'édition en liste passe par la même mutation, la même validation (§6) et la même autorisation (§11) que le formulaire. La liste n'est pas un canal d'écriture privilégié.
Alternatives écartées.

Option B (préférence différentielle) : propagerait les nouveaux défauts, mais complexité forte sur l'ordonnancement exprimé en diff, pour un gain (propagation) jugé non impératif.
Option A brute (sans filtrage) : casse à la première évolution du métamodèle (références mortes) — rejetée.

Point de perf reporté. Une colonne personnalisée pointant vers un attribut d'entité liée (ex. « nom du propriétaire ») retombe sur le pushdown / N+1 du §7 : la personnalisation est un générateur de requêtes dynamiques, à traiter comme tel quand on l'implémentera.