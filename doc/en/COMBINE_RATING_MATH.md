# Combining the ratings — mathematical core

This document does not repeat how each of the two ratings is calculated on its own — that is already done in [CONTRIBUTION_RATING_MATH.md](CONTRIBUTION_RATING_MATH.md) and [WIN_RATING_MATH.md](WIN_RATING_MATH.md). It describes only what sits ON TOP of them: how two independent numbers for the same game are brought together into what a player sees as "their rating", and which ideas and features are specific to this layer.

---

## 0. Idea: one game — two independent numbers

For every game a player plays, two completely independent raw signals are calculated:

- **Contribution** — the result of comparing judge scores against the expectation (all of [CONTRIBUTION_RATING_MATH.md](CONTRIBUTION_RATING_MATH.md)).
- **Win** — the result of comparing the fact of the team's win against the expectation (all of [WIN_RATING_MATH.md](WIN_RATING_MATH.md)).

Neither calculation knows the other exists. But what ends up stored in the player's profile and shown on the leaderboard is not two separate numbers, but several layers that combine them:

```
contribution_*  (CONTRIBUTION_RATING_MATH.md)        win_*  (WIN_RATING_MATH.md)
        │                                                     │
        │                  resolution (section 2)             │
        │          shared trust coefficient for this game     │
        └────────────────────────┬────────────────────────────┘
                                  ▼
                 role_{don, mafia, sheriff, civilian}   ← section 4
               (per-role mix of contribution and win for this game,
                 section 5: "floor/ceiling" protection)
                                  │
                                  │  combine.role_weights.{don,mafia,sheriff,civilian}
                                  │  (one shared set of weights, sections 6 and 7)
                                  ▼
                              overall                        ← section 7

  contribution_* ──┐                                  win_* ──┐
                    │  (same combine.role_weights.*)           │
                    ▼                                          ▼
          contribution_overall                         win_overall      ← section 6
       (standalone, display-only aggregates — not part of overall)
```

This whole document is about what happens on this diagram, not inside `contribution_*`/`win_*`.

---

## 1. Shared input filter

Before either of the two systems calculates a single number, the game passes through a shared filter that is the same for both systems:

- **Excluding whole tournaments/games** — `tournament_exclude_rules` (for example, an administrator has marked a tournament as a duplicate of another one) and `game_exclude_rules` (for example, a game uses an unsupported judging system such as `MR37`/`MR39`/`MR40` from `Mafia Ratings`). Such games are not included in the calculation at all — neither contribution, nor win, nor the games-played counter `games_*` is touched.
- **Validity of a particular game** (`valid_game`) — exactly 10 players with a role line-up of 1 don / 2 mafia / 1 sheriff / 6 civilians, a clear winning team, and the "game broken by scores" rule did not fire: some player on one team got a judge score below a separate "broken game" threshold (not the same one as `normal_game_boundary`), and that very team lost — or such a low-scoring player was found on both teams at once.

---

## 2. Resolution — the shared trust coefficient everything rests on

`resolution` is a number `∈ [0, 1]` defined entirely in [CONTRIBUTION_RATING_MATH.md](CONTRIBUTION_RATING_MATH.md) (section 2, branch A): how finely and informatively the judge separated the players by score in this particular game. It knows nothing about the players' current ratings and nothing about the outcome of the game — only about the distribution of judge scores.

Inside contribution itself this coefficient is already used once — as a pair multiplier (section 5.2 of that document). But the combining layer uses it ONE MORE TIME, at a completely different level: as the shared weight between the WHOLE contribution branch and the WHOLE win branch when the per-role rating is assembled (section 4):

```
coef_contribution_dynamic = resolution
coef_win_dynamic          = 1 − resolution
```

Meaning: if the judge clearly and finely separated the players by score in this game, the combined rating relies mostly on contribution (the scores carry a lot of information). If the scores barely differ (`resolution` close to 0), the combined rating relies almost entirely on the win, because there is little to trust in the judging signal of this particular game, while the outcome of the game is an objective fact regardless of the quality of the judging.

Important: this is one and the same `resolution` number, calculated once per game and reused at two different levels with different meanings — not two values that happen to coincide, but one and the same calculation.

---

## 3. The "full" contribution for the mix — a separate shadow smoothing

It would be natural to simply take the ready, stored contribution delta (`delta_contribution_s`, what is actually added to `contribution_*`) and plug it into the mix with weight `coef_contribution_dynamic`. But that would count `resolution` TWICE: once inside contribution itself (section 5.2 of CONTRIBUTION_RATING_MATH.md, where `resolution` already multiplies the raw pair signal), and a second time here, as `coef_contribution_dynamic`. In games with low resolution, contribution would be suppressed quadratically while win would be suppressed only linearly (`1 − resolution`), which is asymmetric and does not reflect the intent: resolution should weigh the branches against each other EXACTLY once.

So a separate, "full" version of the raw contribution is calculated for the mix — the same pairwise comparisons, the same `contribution_pair_scale`, the same `min` of volatilities, but WITHOUT the internal multiplication by `resolution`:

```
contribution_sum_full(p) = Σ_q≠p  contribution_gain × activity_factor(p, q) × contribution_pair_scale(...) ×
                                    min(m_contribution(p), m_contribution(q)) ×
                                    ( outcome(score_p, score_q) − contribution_expectation(...) )

delta_contribution_full(p) = contribution.k × contribution_sum_full(p)
```

This "full" raw delta then goes through THE SAME smoothing formula as in section 7 of CONTRIBUTION_RATING_MATH.md (gap, dir_match, `tanh` components, `smoothing.dv_max`), from the same current hidden trend of the player — but as a separate, "shadow" calculation whose result is not stored anywhere except in the mix below and is never visible as `contribution_*`.

---

## 4. `role_*` — combined rating per role

```
role_delta_plain(p) = coef_contribution_dynamic × delta_contribution_s_plain_full(p)
                     + coef_win_dynamic          × delta_win_s(p)

role_X = role_X_previous + role_delta_final(p)        (section 5: may be zeroed)
```

Here `delta_win_s(p)` is the ordinary smoothed win delta that is actually stored (section 7 of WIN_RATING_MATH.md) — win needs no separate "shadow" version, because win has no internal multiplication by `resolution` that would need compensating.

`role_don`/`role_mafia`/`role_sheriff`/`role_civilian` are four independent counters per player, one per role; each starts at 1000 and accumulates exactly this mixed delta. This is the only number in the whole system that literally mixes contribution and win into one value on every single game.

---

## 5. "Floor/ceiling" protection

`role_delta_plain` has one additional correction. It is applied before the number goes anywhere further, so it takes effect everywhere `role_delta_final` flows to — `role_*` itself (this section) and, from section 7 on, `overall`; it never affects `contribution_*` and `win_*`, since those are calculated earlier and independently:

```
win_normal_floor_gate    = (combine.win_normal_floor_enabled   && !neutral_contribution_system && actual_win == 1 && my_score ≥ normal_game_boundary)
win_normal_ceiling_gate  = (combine.win_normal_ceiling_enabled && !neutral_contribution_system && actual_win == 0 && my_score ≤ normal_game_boundary)

contribution_floor_effective   = win_normal_floor_gate   && role_delta_plain < 0
contribution_ceiling_effective = win_normal_ceiling_gate && role_delta_plain > 0

role_delta_final = (contribution_floor_effective || contribution_ceiling_effective) ? 0 : role_delta_plain
```

Idea: if the player's team WON and the player played no worse than the norm (`my_score ≥ normal_game_boundary` — the same "normal game" boundary as in CONTRIBUTION_RATING_MATH.md, section 3.2), the combined role rating must not DECREASE — even if pairwise contribution comparisons or table strength would formally produce a negative mix. Symmetrically: if the team LOST and the player played worse than the norm, the role rating must not GROW. In both cases, instead of the negative/positive delta the role simply does not change in this game (`role_delta_final = 0`), rather than being corrected by some other number.

Both gates are additionally switched off entirely in "neutral" games (`neutral_contribution_system`, section 9) — the floor/ceiling is built on `my_score` relative to `normal_game_boundary`, i.e. on trust in the judging system of this particular game. If the judging system in this game is so untrusted that contribution is not calculated from it at all, then conclusions such as "played no worse than the norm"/"played worse than the norm" should not be drawn from the same scores either — so in a neutral game `role_delta_final` is always equal to `role_delta_plain` without any correction.

---

## 6. `win_overall` / `contribution_overall` — role weights within one system (display only)

This is a separate, orthogonal mixing axis — not contribution versus win, but role versus role WITHIN one and the same system:

```
win_overall          = 1000 + Σ_role  combine.role_weights.role × (win_role − 1000)
contribution_overall = 1000 + Σ_role  combine.role_weights.role × (contribution_role − 1000)
```

where the sum runs over all four roles (`don`, `mafia`, `sheriff`, `civilian`) and is recalculated from scratch on every game — even if in this particular game only ONE of the player's four role values changed (in a game they play exactly one role), `win_overall`/`contribution_overall` are still rebuilt from all four current values in full.

There is exactly one set of role weights in the config — `combine.role_weights.{don,mafia,sheriff,civilian}`, currently all equal to `1.0`. It is reused right here for `win_overall` and `contribution_overall`, and once more in section 7 for `overall` — not by coincidence, but because there is deliberately exactly one place where these proportions are listed. If it ever becomes necessary for a role to weigh differently for contribution, for win and for the final rating, that will be three deliberately introduced separate sets, not something already secretly built into the formula for the future.

Important: `win_overall` and `contribution_overall` are standalone, purely display numbers (shown on the leaderboard as separate columns). They are calculated directly from `win_*`/`contribution_*` and do NOT take part in calculating `overall` (section 7) — they are two parallel layers over the same `win_*`/`contribution_*`, independent of each other, not consecutive steps of one pipeline.

---

## 7. `overall` — the single number

`overall` is accumulated in increments directly from the same number that increments `role_*` (section 4) for this game — `role_delta_final`, which is ALREADY weighted by `resolution` (section 2) and has ALREADY passed the "floor/ceiling" protection (section 5). The only thing added on top is the same role weight as in section 6:

```
overall_delta = role_delta_final(p) × combine.role_weights.role

overall = overall_previous + overall_delta
```

This is NOT a rebuild "from scratch" from `win_overall`/`contribution_overall` (section 6) — `overall` has nothing to do with those two numbers (see the note in section 6). It is a direct, cumulative continuation of the same number that was already written to `role_X` for this game, just multiplied once more by the role weight — the SINGLE set of weights from section 6, not a separate one.

Thanks to this, `overall` automatically inherits both properties of `role_delta_final` without recalculating anything:

- resolution is applied to contribution exactly once (section 3), not twice;
- if the floor or ceiling fired in this game (section 5) and `role_X` did not change, `overall` will not change for the same reason either — these are no longer two independent decisions, but one.

There is an exact algebraic identity here, not just a numerical coincidence for particular weight values: since `combine.role_weights.role` is a constant for the role (it does not change from game to game), the accumulation `Σ_games role_delta_final × weight` over all games of one role equals `weight × Σ_games role_delta_final` for that role, i.e. `weight × (role_X − 1000)`. Hence

```
overall = 1000 + Σ_role combine.role_weights.role × (role_role − 1000)
```

— this holds ALWAYS, for any weight values, not only for `1.0`. The difference between `overall` (section 7) and `win_overall`/`contribution_overall` (section 6) is not in the weights — the weights are shared — but in the method of calculation: `win_overall`/`contribution_overall` are rebuilt "from scratch" every game from the current `win_*`/`contribution_*`, while `overall` is accumulated in increments from `role_delta_final`. For this particular linear structure (a constant weight per role) both methods give mathematically the same result — which is exactly why the incremental accumulation of `overall` does not need to be checked against a `win_overall`-style rebuild: they are identical by construction.

---

## 8. `peak_*` — historical maximums

For `overall`, `win_overall`, `contribution_overall` and for every role×system pair (`peak_win_don`, `peak_contribution_mafia`, `peak_role_sheriff`, etc.) the highest value the player has ever reached is stored separately:

```
peak_X = max(peak_X_previous, X_new)
```

This is a purely display value — it affects nothing and is never fed back into the calculation; it only shows the historical peak.

---

## 9. When contribution is silent but win is not

Not every judging system in the historical data is reliable enough to be trusted with the contribution calculation (see CONTRIBUTION_RATING_MATH.md, section 10 — "trust in the judging system as a whole"). There is a separate flag for such games:

```
neutral_contribution_system =
    model_name == 'MAFCLUB_v1'
    || tournament_date < '2018-01-01'
    || (model_name == 'FIIM_v1' && tournament_date ∈ ['2018-01-01', '2022-12-31'])
```

In such games the raw contribution delta is forced to zero (`contribution_sum = 0`, `delta_contribution = 0`) — but this is NOT the same as excluding the game in section 1: the game is still valid, still counted in `games_*`, and `win_*` is calculated for it completely normally, as if nothing special had happened. The only thing that is silent is contribution (and everything that relies on the judge score of this game, including the floor/ceiling from section 5), and it is completely silent: not only the raw delta and the hidden trend are frozen, but also the visible smoothed rating `contribution_*`.

`delta_contribution = 0` alone would not be enough to ensure this. Smoothing (section 7 of CONTRIBUTION_RATING_MATH.md) does not just react directly to the raw delta of this game — it also pulls the visible rating towards the already accumulated gap with the hidden trend (`gap_component`), and that gap could have been non-zero even before the neutral game, left over from some past ordinary games. With `contribution.smoothing.dir_match_required = false` (the current value), `gap_component` is calculated regardless of the sign and value of the current delta — so even with `delta_contribution = 0` the visible rating would keep being pulled towards the old gap, as if this game had said something about the player's contribution, although in fact it had not. That is why `delta_contribution_s_plain`/`delta_contribution_s_plain_full` (what in an ordinary game goes into `contribution_*` and into the "full" version for `role_delta_plain`) are also explicitly zeroed for neutral games — separately from `contribution_sum`, by the same flag:

```
delta_contribution_s_plain      = neutral_contribution_system ? 0 : (ordinary smoothing formula)
delta_contribution_s_plain_full = neutral_contribution_system ? 0 : (ordinary smoothing formula, "full" version)
```

The "full" version would not have affected `role_delta_plain`/`overall` anyway (in a neutral game `coef_contribution_dynamic` is zero anyway — see the zeroing of `resolution` below) — but the standalone `contribution_*` itself, without this second zeroing, would keep drifting ever so slightly in neutral games, contrary to the very meaning of the flag.

The same flag also disables `resolution` (section 2) for this game — it is forced to `0` rather than calculated as usual from the distribution of judge scores:

```
resolution = valid_game && !neutral_contribution_system && n_players > 1 && score_groups > 1
                ? score_resolution(...)
                : 0
```

Meaning: in a neutral game contribution does not take part at all, so there is no point in reserving part of the mix weight for it (section 2) — `coef_contribution_dynamic = resolution = 0`, `coef_win_dynamic = 1 − resolution = 1`, and the whole mix `role_delta_plain`/`overall_delta` in this game relies only on win, without wasting part of the weight. Without this condition `resolution` could turn out non-zero simply because old/untrusted judge scores formally distinguish something — and then win would get less weight than it should, even though contribution would still not add a single point of signal.

The same flag also switches off the "floor/ceiling" protection entirely (section 5) — it, too, is built on the judge score of this game relative to `normal_game_boundary`, and therefore requires the same trust in the judging system as contribution itself.

This is a deliberate asymmetry in the capabilities of the two systems: win needs only the outcome of the game (which is always known), so it can evaluate the entire historical game database uniformly; contribution needs a trusted judging methodology, so it is active only where one exists. It is precisely because these are two independent systems rather than one shared formula that missing or unreliable judging data does not throw the whole game out of the rating — only one of the two signals is lost, not both at once.

---

## 10. Calibration: summary map of combine-layer parameters

| Parameter | Section | Which mechanism it reproduces |
|---|---|---|
| `combine.win_normal_floor_enabled` | 5 | Enables the protection: win + game no worse than the norm ⇒ `role_*` (and, through it, `overall`) does not decrease (except in neutral games, section 9) |
| `combine.win_normal_ceiling_enabled` | 5 | Enables the protection: loss + game worse than the norm ⇒ `role_*` (and, through it, `overall`) does not increase (except in neutral games, section 9) |
| `combine.role_weights.{don,mafia,sheriff,civilian}` | 6, 7 | The single set of role weights — used in the same way when assembling `win_overall`, `contribution_overall` (section 6) and when accumulating `overall` (section 7) |
| `shared.first_killed_factor` | — | Literally one and the same parameter used by BOTH systems inside their `activity_factor` — not two copies of one value, but one shared config object |

`resolution` (section 2) is itself calibrated inside contribution (`entropy_power`, `group_power` — CONTRIBUTION_RATING_MATH.md, section 8) — here, in the combine layer, it is already a ready number with no settings of its own at this level.

It is worth noting separately: `win.k`/`contribution.k` and `win.table_strength_eps`/`contribution.table_strength_eps` currently match numerically (`6` and `900` respectively) — but these are not shared parameters; they are two independently calibrated values that at the moment simply happen to be tuned to the same values. Unlike `shared.first_killed_factor`, this is a coincidence, not sharing.

---

## 11. Caveats

Below are several deliberate features of the combining layer specifically that are worth keeping in mind when interpreting `role_*`/`overall`.

- **`role_*` and `overall` are not zero-sum in any sense.** The zero-sum-per-table proofs in both documents (section 6 in each) apply strictly to the raw delta and hidden trend of EACH system separately — and both already state explicitly that the visible, smoothed rating does not have this property (the second-to-last caveat in both documents). The combine layer is built on top of visible numbers that are already not zero-sum (`delta_win_s`, the "shadow" smoothed version of contribution), and by itself it adds no new balance and does not try to.
- **Role weight is one shared set, not three.** `combine.role_weights.{don,mafia,sheriff,civilian}` (section 6) is the only place in the config where role proportions are listed, and it is used in the same way for `win_overall`, `contribution_overall` (section 6) and for accumulating `overall` (section 7). This is a deliberate simplification: there are no separate weight sets "for contribution", "for win" and "for the total" until a concrete reason appears for them to diverge. Currently all four weights are equal to `1.0`.
- **`overall = 1000 + Σ_role weight.role × (role_role − 1000)` is an exact identity, not an observation for particular weights.** The detailed derivation is in section 7: since the role weight is constant, the incremental accumulation of `overall` and a hypothetical rebuild "from scratch" in the style of `win_overall`/`contribution_overall` give the same result for ANY weights, not only `1.0`. In other words, `overall` can be read both as "accumulated history" and as "an instant snapshot of the current `role_*`" — these are not two different numbers that happened to match, but one and the same number calculated in two ways.
- **This layer, too, is the result of observation, not the only possible design.** As in each of the two documents separately, the choice of this particular way of combining (resolution as the shared weight, "shadow" parallel smoothing of contribution, floor/ceiling specifically for `role_*`/`overall`, a single set of equal role weights) is not a mathematical necessity but the current working decision, which is worth revisiting as new observations accumulate on how `role_*`/`overall` behave in real tournaments.
