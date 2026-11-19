---
name: fit-suivi
description: Étape 3 de la suite Fit : check-in hebdo qui lit l'onglet Suivi de l'artefact, analyse poids, taille, perfs et adhérence, puis ajuste les kcal ou relance fit-bilan.
---

# Fit · Étape 3 — Suivi hebdomadaire

## Prérequis (artefact « Fit — Fiche profil »)
`list_artifacts` query « Fit — Fiche profil » → slug. Lire `profil/current`, `resultats/bilan`, `resultats/plan`, `resultats/journal` (db_get) et tous les `checkins` (db_list).
- Plan absent → lancer fit-plan.
- Check-in de la semaine courante absent ou < 3 pesées → demander de remplir l'onglet Suivi, s'arrêter.

## 1. Calculs
- Moyenne des `pesees` de la semaine. Tendance = variation en % entre les 2 dernières moyennes hebdo.
- Après ≥ 3 semaines avec `kcal_moy` : TDEE réel = kcal moyennes − (variation de poids en kg par jour × 7700). Écrire `tdee_reel` dans `resultats/bilan`.
- Mettre à jour `poids_kg` et `tour_taille_cm` de `profil/current` avec les dernières valeurs.

## 2. Décision (dans l'ordre, s'arrêter à la première qui s'applique)
1. Case `alertes` cochée, ou perte > 1,5 %/sem deux semaines de suite → **CONSULTER** : arrêter le déficit, revenir à la maintenance, orienter vers un pro.
2. `adherence_pct` < 80 → **MAINTENIR** les kcal ; trouver ce qui bloque et simplifier (renvoyer à fit-plan).
3. Perfs en baisse 2 semaines + `energie` ou `sommeil` ≤ 2 → **DIET BREAK** : 1–2 semaines à maintenance.
4. Fin de phase → **CHANGER DE PHASE → relancer fit-bilan** :
   - Sèche : cible de gras atteinte ou 12–16 sem depuis `debut_phase` (1–2 sem de maintenance d'abord)
   - Masse : bf ≥ 18 % (H) / 28 % (F) ou tour de taille > +4 cm depuis le début
   - Recompo : 12 semaines écoulées
   - Plateau de 3 semaines malgré ≥ 90 % d'adhérence et 2 ajustements
5. Écart au rythme cible 2 semaines de suite → **AJUSTER** (glucides/lipides, jamais les protéines) :
   - Sèche : perte < 0,5 %/sem → −150 kcal ; > 1,2 %/sem → +150 kcal
   - Masse : gain sous la fourchette → +150 ; au-dessus, ou taille > +1 cm/mois sans progrès des perfs → −150
   - Recompo : taille en hausse → −100 ; perfs en baisse 3 semaines → +100
6. Sinon → **MAINTENIR**.

## 3. Application
- `AJUSTER` / `DIET BREAK` : mettre à jour `cibles` et `markdown` de `resultats/plan` (portions réajustées, vérification ±5 % refaite), puis `resultats/courses` si besoin.
- `CHANGER DE PHASE` : enchaîner sur fit-bilan avec la fiche à jour.

## Sortie : db_write `resultats/journal`
Ajouter une entrée au tableau `lignes` {semaine, poids_moy, delta_pct, taille, adherence, perfs, energie, kcal_cible, verdict, raison} et régénérer `markdown` :
```
| Sem | Poids moy | Δ % | Taille | Adhérence | Perfs | Énergie | Kcal cible | Verdict |
```
Répondre en 2–3 lignes : décision, règle appliquée, objectif de la semaine suivante.
