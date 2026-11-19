# skill-fitness

Suite de 3 skills Claude chaînées pour analyser sa situation et suivre un plan nutritionnel visant à devenir fit et musclé. Même principe que [founder-skill](https://github.com/Jakeschincariol/founder-skill) : chaque étape lit ce que la précédente a écrit, et les décisions sont calculées à partir des chiffres.

```
fit-bilan → fit-plan → fit-suivi (chaque semaine) ─┐
    ▲                                              │
    └────────── changement de phase ───────────────┘
```

| Skill | Rôle | Écrit |
|---|---|---|
| `fit-bilan` | Crée ou lit la fiche profil, garde-fous santé, BMR/TDEE, % de gras, décision de phase (sèche / recompo / prise de masse) | `resultats/bilan` |
| `fit-plan` | Macros, structure des repas, journées types vérifiées (±5 % kcal), échanges, courses, suppléments | `resultats/plan`, `resultats/courses` |
| `fit-suivi` | Check-in hebdo : tendance de poids, tour de taille, perfs, adhérence → maintenir, ajuster, diet break, changer de phase | `resultats/journal` |

## Source de vérité : l'artefact « Fit — Fiche profil »

Au premier lancement, `fit-bilan` crée un artefact claude.ai avec 3 onglets :
- **Profil** : formulaire à remplir (mensurations, entraînement, activité, alimentation, budget, garde-fous)
- **Suivi** : check-in de la semaine (pesées, tour de taille, perfs, énergie)
- **Résultats** : bilan, plan, courses et journal générés par les skills

Les données sont stockées dans la base de l'artefact et persistent d'une conversation à l'autre.

## Installation

- **claude.ai** : Paramètres → Capacités → Skills → importer chaque dossier de `skills/` (zippé).
- **Claude Code** : `cp -r skills/* ~/.claude/skills/`

## Utilisation

1. « Lance fit-bilan » → remplir la fiche → « c'est rempli »
2. « Lance fit-plan »
3. Chaque semaine : remplir l'onglet Suivi → « Lance fit-suivi »

> Outil d'information, pas un avis médical.
>>>>>>> 0c3f60d (Ajoute la suite de skills Fit (bilan, plan, suivi))
