---
name: fit-bilan
description: Étape 1 de la suite Fit : crée ou lit l'artefact « Fit — Fiche profil », calcule le TDEE et décide la phase (sèche, recompo, masse). Lancer en premier.
---

# Fit · Étape 1 — Bilan

Suite chaînée : **fit-bilan → fit-plan → fit-suivi (hebdo) → retour à fit-bilan au changement de phase**. Chaque étape lit ce que la précédente a écrit. Les décisions sont **calculées** (exécuter le code), jamais estimées de tête.

## Source de vérité : l'artefact « Fit — Fiche profil »
Toutes les données vivent dans la base (`db`) de cet artefact et persistent d'une conversation à l'autre. Lire avec db_get / db_list, écrire avec db_write (slug de l'artefact). Aucun fichier local.

### Trouver ou créer
1. `list_artifacts` avec query « Fit — Fiche profil ». Trouvé → garder son slug.
2. Sinon (premier lancement) : lire `guide` (topic pages), écrire la page décrite ci-dessous, la publier (`publish_version`, title « Fit — Fiche profil », icon « form »), puis `declare_capabilities` :
```json
{"db":{"rules":[{"path":"profil","read":"view","write":"owner"},{"path":"checkins","read":"view","write":"owner"},{"path":"resultats","read":"view","write":"owner"}]},"user":{}}
```
Dire à l'utilisateur de remplir l'onglet Profil, d'appuyer sur Enregistrer et de revenir dire « c'est rempli ». S'arrêter là.

### Documents de la base
- `profil/current` : rempli par l'utilisateur (onglet Profil)
- `checkins/<AAAA-Snn>` : check-in hebdo rempli par l'utilisateur (onglet Suivi)
- `resultats/bilan`, `resultats/plan`, `resultats/courses`, `resultats/journal` : écrits par les skills, chacun avec `markdown` et `updated_at` ; affichés dans l'onglet Résultats

### Page à construire (une page HTML autonome, pensée mobile)
- **Profil** : formulaire pré-rempli depuis `profil/current`, bouton Enregistrer (écrit le doc entier + `updated_at`), confirmation « Enregistré ✓ <date> ». Champs :
  - Identité : `sexe` (H/F), `age`, `taille_cm`, `poids_kg` (moyenne de pesées du matin)
  - Mensurations (optionnel) : `tour_taille_cm` (nombril), `cou_cm`, `hanches_cm`, `bf_pct` + `bf_methode`
  - Entraînement : `annees_muscu`, `seances_sem`, `duree_seance_min`
  - Activité : `pas_jour`, `metier` (assis/debout/physique), `autres_sports_h_sem`
  - Objectif : `objectif`, `horizon_mois`, `priorite` (esthétique/force/santé)
  - Alimentation : `journee_type`, `nb_repas`, `regime`, `aime`, `deteste`, `budget_sem`, `temps_cuisine_min`, `batch_cooking` (oui/non)
  - Garde-fous (cases) : `moins_18`, `grossesse_allaitement`, `maladie_ou_medicament`, `antecedents_trouble_alimentaire`
  - `notes`
- **Suivi** : formulaire de la semaine ISO courante (doc `checkins/<AAAA-Snn>`) : `pesees` (7 champs, un par jour), `tour_taille_cm`, `adherence_pct`, `kcal_moy` (optionnel), `seances_faites`, `perfs` (texte : exercice charge × reps), `faim`, `energie`, `sommeil` (1–5), `alertes` (cases : vertiges, cycle perturbé, obsession/compulsions), `notes`.
- **Résultats** : affiche en lecture seule le markdown de `resultats/bilan`, `resultats/plan`, `resultats/courses` et `resultats/journal` ; « Pas encore généré » si vide.
- Technique : `const db = await claude.use("db")` ; si `null` → bandeau « Ouvre cette page connecté à claude.ai pour enregistrer » ; un seul `onSnapshot` par doc ; minuterie de chargement de 10 s ; mini-rendu markdown maison (titres, listes, tableaux, gras) ; aucune donnée codée en dur.

## 0. Lire le profil
db_get `profil/current`.
- Absent → renvoyer vers l'artefact, s'arrêter.
- Requis manquants (`sexe`, `age`, `taille_cm`, `poids_kg`, `seances_sem`, `pas_jour` ou `metier`) → verdict `À COMPLÉTER` : lister les champs à remplir dans la fiche.
- Ne jamais redemander en chat une info déjà dans la fiche. Une info donnée en chat → la reporter dans `profil/current` pour garder une seule source.

## 1. Garde-fous
Une case garde-fou cochée, ou IMC < 18,5 avec objectif de perte → verdict `CONSULTER` : aucune cible calorique chiffrée, orienter vers un médecin ou un nutritionniste, écrire `resultats/bilan` avec ce verdict, s'arrêter.

## 2. Calculs
```python
import math
def bilan(sexe, age, taille, poids, ventre=None, cou=None, hanches=None, bf=None, facteur=1.45):
    if bf is None and ventre and cou:
        if sexe == "H":
            bf = 495/(1.0324-0.19077*math.log10(ventre-cou)+0.15456*math.log10(taille))-450
        elif hanches:
            bf = 495/(1.29579-0.35004*math.log10(ventre+hanches-cou)+0.22100*math.log10(taille))-450
    bmr = 10*poids + 6.25*taille - 5*age + (5 if sexe == "H" else -161)   # Mifflin-St Jeor
    if bf is not None:
        bmr = (bmr + 370 + 21.6*poids*(1-bf/100)) / 2                     # moyenne avec Katch-McArdle
    return dict(bf=bf, mm=poids*(1-bf/100) if bf else None, bmr=round(bmr),
                tdee=round(bmr*facteur), ratio_taille=ventre/taille if ventre else None)
```
Facteur d'activité (entraînement inclus) : 1.3 sédentaire + ≤ 3 séances · 1.45 ~7k pas + 3–4 séances · 1.6 ~10k pas + 4–5 séances · 1.75 métier physique ou > 6 h de sport/sem.
Si `resultats/bilan` contient déjà un `tdee_reel` (recalibré par fit-suivi), l'utiliser à la place.

## 3. Décision de phase (dans l'ordre, seuils H / F en % de gras)
1. bf ≥ 22 / 32 → **SÈCHE**
2. Débutant (< 1 an) : bf ≥ 15 / 25 → **RECOMPO**, sinon **PRISE DE MASSE**
3. bf ≤ 15 / 25 → **PRISE DE MASSE**
4. Zone intermédiaire, non débutant → **SÈCHE** courte si priorité esthétique, sinon **RECOMPO**

Si bf inconnu : ratio tour de taille / taille ≥ 0,53 → sèche ; ≤ 0,45 → masse ; sinon recompo.

Rythme et cible :
- **Sèche** : −0,5 à −1 % du poids/sem. Déficit/j = poids × rythme × 7700 / 7, plafonné à 25 % du TDEE. Durée 8–16 sem.
- **Masse** : débutant +1 à 1,5 %/mois, intermédiaire +0,5 à 1 %, avancé +0,25 à 0,5 %. Surplus/j = poids × rythme × 7700 / 30. Durée 4–9 mois.
- **Recompo** : maintenance à −10 %. Réévaluer à 12 sem.
- Plancher : jamais sous le BMR ni sous 1500 kcal (H) / 1200 kcal (F).

## 4. Verdict (calculé)
`PRÊT` (→ fit-plan) · `À COMPLÉTER` (champs manquants) · `CONSULTER` (garde-fou).

## Sortie : db_write `resultats/bilan`
Champs : `verdict`, `phase`, `debut_phase` (date), `duree_sem`, `rythme_pct_sem`, `bmr`, `tdee`, `cible_kcal`, `bf_pct`, `masse_maigre_kg`, `updated_at`, et `markdown` :
```
# Bilan — <date>
Verdict : PRÊT
Phase : SÈCHE | 12 sem | −0,7 %/sem (−0,6 kg/sem)
BMR : … | TDEE : … (facteur …) | Cible : … kcal/j
% gras : … (méthode) | Masse maigre : … kg
Règle appliquée : <n° + 1 ligne>
À vérifier en suivi : <facteur d'activité, mesure du gras…>
```
Terminer par : « Résultat visible dans l'onglet Résultats de ta fiche. Étape suivante : fit-plan. » Rappeler une fois que c'est de l'information, pas un avis médical.
