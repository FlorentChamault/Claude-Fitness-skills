---
name: fit-plan
description: Étape 2 de la suite Fit : lit la fiche profil et le bilan dans l'artefact, construit macros, journées de repas vérifiées et liste de courses. Après fit-bilan.
---

# Fit · Étape 2 — Plan nutritionnel

## Prérequis (artefact « Fit — Fiche profil »)
`list_artifacts` query « Fit — Fiche profil » → slug. Puis db_get `profil/current` et `resultats/bilan`.
- Artefact ou bilan absent, ou verdict `À COMPLÉTER` → lancer fit-bilan.
- Verdict `CONSULTER` → s'arrêter, aucun chiffre.
Toutes les contraintes viennent de la fiche (`regime`, `aime`, `deteste`, `budget_sem`, `temps_cuisine_min`, `batch_cooking`, `nb_repas`). Ne pas les redemander ; une précision donnée en chat → la reporter dans `profil/current`.

## 1. Macros (calculer, selon `phase` et `cible_kcal`)
- **Protéines** : sèche 2,0–2,2 g/kg ; masse / recompo 1,6–2,0 g/kg. Si bf ≥ 25 % (H) / 35 % (F) : 2,4 g/kg de masse maigre.
- **Lipides** : 0,7–1,0 g/kg, jamais < 20 % des kcal (minimum 0,5 g/kg en sèche dure).
- **Glucides** : le reste. Si < 2 g/kg avec ≥ 4 séances/sem → descendre d'abord les lipides vers leur plancher.
- Fibres : 14 g / 1000 kcal. Eau : ~35 ml/kg + 500 ml par heure d'entraînement.

## 2. Structure
- 3–5 repas (selon `nb_repas`), 0,3–0,5 g/kg de protéines chacun (≥ 25–40 g), espacés de 3–5 h.
- 1–3 h avant la séance : glucides + protéines, peu de gras et de fibres. Après : protéines + glucides.
- Plus de glucides autour des séances. Mêmes kcal les jours de repos sauf demande.

## 3. Journées de repas
- 2–3 journées types en rotation + tableau d'échanges (ex. 150 g poulet ⇄ 200 g skyr ⇄ 3 œufs + 150 g blancs).
- Grammes, cru ou cuit précisé. Valeurs issues de tables fiables (Fichier canadien sur les éléments nutritifs, Ciqual, USDA).

## 4. Vérification (calculée, bloquante)
Totaliser chaque journée. Valide si kcal à ±5 % de la cible et protéines à ±10 g. Sinon ajuster les portions et recalculer.

## 5. Courses
Liste hebdo par rayon, quantités totales, coût estimé vs `budget_sem`. Dépassement → substitutions économiques (œufs, lentilles, cuisses de poulet, thon, yogourt grec, avoine, riz, protéine en poudre), puis refaire l'étape 4.

## 6. Suppléments (preuves solides seulement)
Créatine monohydrate 3–5 g/j · protéine en poudre si l'objectif protéines est dur à atteindre · vitamine D si peu de soleil (dose à valider avec un médecin) · caféine optionnelle avant la séance. Rien d'autre sans raison.

## Verdict
`PLAN VALIDÉ` (étape 4 passée partout) · `À AJUSTER` (préciser l'écart).

## Sorties (db_write)
- `resultats/plan` : `verdict`, `cibles` {kcal, proteines_g, lipides_g, glucides_g, fibres_g, eau_l}, `updated_at`, `markdown` (cibles, structure, journées types avec totaux, échanges, suppléments)
- `resultats/courses` : `cout_estime`, `updated_at`, `markdown` (liste par rayon)

Faire valider ; toute modification → refaire l'étape 4 et réécrire. Terminer par : « Plan dans l'onglet Résultats. Pèse-toi chaque matin et remplis l'onglet Suivi ; fit-suivi chaque semaine. »
