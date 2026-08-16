---
name: batch-cooking
description: >
  Gère la planification hebdomadaire du batch cooking, les recettes et la liste de courses
  pour ce foyer (2 personnes, sportifs). Utiliser cette skill dès que l'utilisateur veut créer
  ou modifier le planning de la semaine, ajouter/changer/retirer une recette, régénérer la liste
  de courses, ou demande plus de détails sur une recette déjà proposée (ingrédients, étapes,
  nutrition, coût) — même formulé de façon informelle : qu'est-ce qu'on mange cette semaine,
  remplace le plat de jeudi, donne-moi la recette complète du gratin, la liste pour la semaine
  prochaine, ajoute une recette à la base. Toujours consulter cette skill avant de travailler
  sur ces sujets plutôt que de repartir de zéro.
---

# Batch Cooking — Planification, recettes & courses

## Contexte fixe du foyer
- 2 personnes, **sportives** (portions et nutrition ajustées en conséquence : viser ~550-650 kcal et ~28-45g de protéines par portion de dîner, quitte à booster une recette avec un œuf dur, une portion de viande/poisson plus généreuse, ou des légumineuses en accompagnement).
- Aucune contrainte alimentaire (pas d'allergie, pas de régime spécifique).
- Seuls les **dîners de lundi, mardi, mercredi, jeudi** sont couverts (4 repas/semaine).
- **Cuisson unique le dimanche matin**, en une session de batch cooking.
- Conservation **exclusivement au frais en Tupperware** — jamais de congélation.
- Recettes : mix entre recettes personnelles de l'utilisateur et suggestions de Claude.

## Sources de vérité — bases Notion
- Page conteneur "🍲 Batch Cooking" : https://app.notion.com/p/3ba97b77e7a6811eb546e7ed918f6e73
- Base **Recettes** : https://app.notion.com/p/d60b4737830840e88b806638fa9294f1
  Champs : Nom, Catégorie, Type de protéine, Portions de base, Temps de préparation (min), Durée de conservation (frigo, jours), Ingrédients, Instructions, Calories/Valeur énergétique/Protéines/Glucides/Lipides (par portion), Coût estimé (€), Source, Tags. Vue "Galerie" disponible (sans photo de couverture — décision prise).
- Base **Planning hebdo** : https://app.notion.com/p/9067759e4a4646f08ba2fb53e565164e
  Champs : Semaine, Recette (relation), Jour de conso (Lundi/Mardi/Mercredi/Jeudi), Portions prévues, Statut, Notes.

Toujours lire ces bases via les outils Notion avant d'agir (ne pas halluciner leur contenu). Utiliser `notion-fetch` / `notion-query-data-sources` pour consulter, `notion-create-pages` / `notion-update-page` pour modifier.

## Règle de conservation (à respecter à chaque planification)
Le décalage entre la cuisson (dimanche) et la consommation va de J+1 (lundi) à J+4 (jeudi). Chaque recette retenue pour un jour donné doit avoir une **durée de conservation au frigo ≥ nombre de jours jusqu'à sa consommation**. En pratique : recettes fragiles (tag "fragile", poisson, œuf, crudités) → lundi/mardi ; recettes qui tiennent bien (tag "se bonifie", mijotés, gratins, légumineuses) → mercredi/jeudi.

## Règle anti-répétition
Avant de proposer les 4 recettes de la semaine, consulter l'entrée la plus récente de **Planning hebdo** (la semaine immédiatement précédente) et **exclure les 4 recettes qui y figurent déjà** — ne jamais reproposer telle quelle une recette utilisée la semaine passée. Au-delà d'une semaine, la variété reste souhaitable mais n'est pas une contrainte stricte.

## Semaines particulières (vacances, absence, imprévu)
Aucune règle automatique : ne pas essayer de détecter ou anticiper ces cas. Si la semaine n'est pas standard (moins de 4 dîners, absence, vacances...), c'est l'utilisateur qui le précise au moment de la demande de planification — ajuster simplement le nombre de jours/recettes en fonction de ce qu'il indique à ce moment-là.

---

## Calcul des dates
Avant toute planification, appeler l'outil de date/heure actuelle (`user_time_v0`) — ne jamais déduire "la semaine prochaine" ou "dimanche" de mémoire ou d'une date vue plus tôt dans la conversation. À partir de la date du jour, calculer le **prochain dimanche** (jour de cuisson) puis le lundi, mardi, mercredi, jeudi qui suivent (jour de conso). Si l'utilisateur précise une autre semaine ("dans 15 jours", "la semaine du 24"), ajuster le calcul en conséquence mais toujours à partir de la date réelle du jour, jamais d'une date supposée.

## Workflow 1 — Créer ou modifier la planification de la semaine
1. Choisir 4 recettes dans la base **Recettes** (une par jour lundi→jeudi), en respectant la règle de conservation ci-dessus et en variant par rapport aux semaines précédentes (regarder les dernières entrées de "Planning hebdo").
2. Présenter la proposition à l'utilisateur (recette, jour, conservation, nutrition, coût) et laisser confirmer ou ajuster avant de créer quoi que ce soit.
3. Une fois validé, créer/mettre à jour les 4 entrées dans **Planning hebdo** (Semaine, Recette, Jour de conso, Portions prévues, Statut = "Pas commencé").
   - **Portions prévues** = 2 (le foyer), quel que soit le nombre de "Portions de base" de la recette. Si "Portions de base" > 2 (ex. cake en 6 parts, tarte en 4 parts), le signaler à l'utilisateur : soit on cuisine la recette telle quelle et il y aura des portions en plus (à consommer en extra ou sur un autre repas), soit on réduit les quantités d'ingrédients au prorata pour ne faire que 2 portions. Demander la préférence si ce n'est pas déjà précisé, plutôt que de choisir silencieusement.
4. Enchaîner naturellement sur la génération de la liste de courses (Workflow 3), sauf si l'utilisateur ne le demande pas.

Pour modifier une semaine déjà planifiée (remplacer un plat, changer un jour) : mettre à jour l'entrée correspondante dans Planning hebdo plutôt que d'en recréer une nouvelle.

## Workflow 2 — Ajouter, changer ou retirer une recette de la base
- **Ajout** : demander/déduire ingrédients, temps, température de cuisson. Estimer conservation frigo, nutrition (ajustée sportif) et coût si non fournis, en le signalant clairement comme une estimation. Créer la page dans Recettes avec tous les champs renseignés.
- **Modification** : mettre à jour uniquement les champs concernés via `notion-update-page`.
- Toujours garder une base suffisamment variée en recettes "fragiles" (2j) et "se bonifie" (4j) pour pouvoir couvrir les 4 jours chaque semaine.

## Workflow 3 — Générer la liste de courses (livrable PDF)
- Partir des 4 recettes du Planning hebdo de la semaine concernée.
- Agréger les ingrédients, regrouper les doublons. **Générer la liste comme si le stock était à zéro** (pas de gestion de placard côté Claude) — c'est l'utilisateur qui indique ensuite ce qu'il faut retirer avant l'achat.
- Structurer par rayon : Fruits & légumes / Viandes & poissons / Crémerie & œufs / Épicerie (+ Surgelés/Boissons si besoin).
- Livrable systématique : un **PDF au format A4 paysage** avec :
  - **Page 1** : planning de la semaine (fiches par jour : recette, kcal, protéines, coût, conservation) + liste de courses façon ticket de caisse par rayon + coût total estimé + liens vers les pages Notion Planning hebdo et Recettes.
  - **Page 2** : préparation et cuisson détaillées, étape par étape, pour chacune des 4 recettes (temps de prépa, température/durée de cuisson en évidence).
  - Voir `references/pdf-template.html` pour le gabarit HTML/CSS à réutiliser et adapter (couleurs, polices, structure des tickets et fiches). Générer le HTML rempli avec les données de la semaine, puis convertir en PDF avec `wkhtmltopdf` :
    ```
    wkhtmltopdf --enable-local-file-access --page-size A4 --orientation Landscape \
      --margin-top 0 --margin-bottom 0 --margin-left 0 --margin-right 0 \
      semaine.html semaine.pdf
    ```
  - Utiliser uniquement des polices locales (DejaVu Sans/Serif/Mono, Liberation) : pas d'accès réseau garanti dans l'environnement d'exécution, donc pas de Google Fonts ni de CDN.
  - Toujours vérifier le rendu (conversion en image avec `pdftoppm -png` puis `view`) avant de livrer.
  - **Nom de fichier** : `semaine-JJ-MM.pdf`, où JJ-MM est la date du lundi de la semaine planifiée (ex. semaine du 17 au 20 août → `semaine-17-08.pdf`).
- Envoyer aussi la liste dans **Notion**, en checklist cochable (voir ci-dessous) — plus dans Reminders (abandonné).
- Présenter le fichier final avec `present_files`.

### Liste de courses dans Notion
En plus du PDF, créer (ou mettre à jour si elle existe déjà) une **page Notion checklist**, enfant de la page conteneur "🍲 Batch Cooking" (https://app.notion.com/p/3ba97b77e7a6811eb546e7ed918f6e73), intitulée `🛒 Courses — semaine du JJ/MM`. Contenu en Markdown Notion : un titre `##` par rayon, puis les produits en cases à cocher `- [ ] produit (quantité)`, et une ligne de total estimé en bas. Cette page Notion est le livrable de référence pour la liste de courses, que la demande arrive **ici en conversation ou via la génération du PDF** — les deux déclenchent la création/mise à jour de cette page, en plus du PDF lui-même.

## Workflow 4 — Détails sur une recette déjà proposée
Quand l'utilisateur demande plus de détails sur une recette évoquée précédemment (ingrédients complets, étapes, nutrition, coût, conservation) :
1. Aller chercher la fiche à jour dans Notion (`notion-fetch` sur la page Recettes concernée) plutôt que de se fier à la mémoire de la conversation — la recette a pu être modifiée depuis.
2. Présenter : ingrédients avec quantités, étapes de préparation puis de cuisson (température/durée si applicable), nutrition par portion, coût estimé, durée de conservation.
3. Si l'utilisateur ajuste la recette pendant l'échange (quantité, ingrédient remplacé...), proposer de mettre à jour la fiche Notion en conséquence.

---

## Décisions déjà actées (ne pas rouvrir la discussion, juste appliquer)
- Pas de congélation, conservation au frais en Tupperware uniquement.
- Pas de photo de couverture sur les fiches recettes Notion.
- Reminders : abandonné pour la liste de courses — remplacé par une page Notion checklist (voir Workflow 3).
- App Notes iPhone : pas de raccourci pour convertir du texte en checklist automatiquement — hors sujet pour ce skill, le livrable est le PDF.
- Gestion de stock : aucune, liste toujours générée à zéro, retraits gérés par l'utilisateur après coup.
- Budget : champ "Coût estimé (€)" rempli par estimation de Claude, sans objectif chiffré à respecter — juste de la visibilité.
- Automatisation (tâche planifiée hebdomadaire Cowork, courses en drive via Claude in Chrome) : hors du périmètre de ce skill, gérées séparément par l'utilisateur.

## Documents de référence complémentaires
Si disponibles dans les fichiers du projet : `spec-batch-cooking.md` (spec métier/technique complète) et `contexte-projet-batch-cooking.md` (état d'avancement, éléments encore ouverts). Les consulter si présents pour du contexte supplémentaire, sans dupliquer leur contenu ici.
