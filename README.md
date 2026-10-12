**English** | [Русский](README.ru.md) | [Українська](README.uk.md)

# Open Mafia Skill

Open Mafia Skill is an open rating system for sports Mafia.

## What it is for

The system estimates a player's skill from tournament results. It uses the sports Mafia format: a table of 10 players (1 don, 2 mafia, 1 sheriff, 6 civilians), a team result for every game and an individual score from the judge.

Every game is evaluated from two independent angles:

- **how well the player played** — judged by the scores the judge gave, compared with the other players at the table;
- **whether their team won** — compared with how likely that win was, given the strength of the players and teams.

Both signals are compared with what the player's current rating predicted, so a strong player gains less from beating weak opponents, and a weak player gains more from an unexpected success.

The system works with games judged under different judging systems (`iMafia`, `FIIM`, `EG`, `CLM`, `MafGame`, …) and calibrates for each of them separately. If a judging system is not reliable for some period, its judge scores are ignored and only the game result is used.

## What it consists of

| Part | What it does | Document |
|---|---|---|
| **Contribution** rating | Measures the quality of play from judge scores: pairwise comparison with every player at the table, weighted by how detailed the judging was in that game | [CONTRIBUTION_RATING_MATH.md](doc/en/CONTRIBUTION_RATING_MATH.md) |
| **Win** rating | Measures team results relative to the expectation: role asymmetry (blacks vs reds), personal and team strength | [WIN_RATING_MATH.md](doc/en/WIN_RATING_MATH.md) |
| **Combining** | Merges the two into per-role ratings (don, mafia, sheriff, civilian) and a single `overall` rating | [COMBINE_RATING_MATH.md](doc/en/COMBINE_RATING_MATH.md) |
| Configuration | Engine parameters and per-judging-system calibration: [`global_params.json`](config/global_params.json), [`seasons_params.json`](config/seasons_params.json) | [config/](config/) |

The documentation is available in three languages: [English](doc/en/), [Русский](doc/ru/), [Українська](doc/uk/).

## License

Licensed under the [Apache License 2.0](LICENSE). See [NOTICE](NOTICE) for authorship.
