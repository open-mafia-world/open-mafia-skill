# "Win" rating — mathematical core

The "Win" rating measures whether the player's team won or lost a particular game and compares this fact with what was expected of them: their strength relative to the opponents and the strength of their team relative to the opposing team. The judge score plays no part in this calculation at all — only the outcome of the game and the participants' current ratings.

---

## 0. Idea: a win is a team fact compared with the expectation

For each game every player has exactly one fact — whether their team won or not (`actual_win ∈ {0, 1}`, the same value for all players on one team). This fact is compared with an expectation that is made up of three independent sources:

1. **Role asymmetry** — the blacks (don + mafia, 3 players) and the reds (sheriff + civilians, 7 players) are fundamentally unequal: these are roles that differ by the very rules and mechanics of the game, not just a different number of players on a team. So an "even match" for them is not 50/50 but an empirical value that can be tracked through the black team's win rate over the last N games (for example, over the last 1000 games) — this way the asymmetry stays current rather than being fixed once and for all (`p_black_win`).
2. **Personal rating difference** — how much higher or lower the player's rating is than that of a particular opponent from the other team.
3. **Team strength difference** — how much stronger or weaker by rating the player's whole team is, on average, compared with the opposing team.

```
   actual_win ∈ {0, 1}                     win expectation
   (fact: did my team win)                 (win_expectation)
            │                                      ▲
            │                      ┌────────────────┼────────────────┐
            │                      │                │                │
            │               role asymmetry    personal rating   team strength
            │               (p_black_win)        difference       difference
            │                      │                │                │
            │                      └────────────────┼────────────────┘
            │                                        │
            └──────────────────────┬─────────────────┘
                                    ▼
                      pairwise comparison (section 5):
                 actual_win minus what was expected of this pair
```

All three sources of expectation are combined into one number — `win_expectation` — and then this number is compared with the actual outcome for each "player — opponent from the other team" pair. The difference between the fact and the expectation, summed over all opponents, is the raw signal of the change in the "Win" rating for this game.

---

## 1. Input data

The input is the team line-up and the outcome of the game:

```
blacks = {don, mafia}            (3 players)
reds   = {sheriff, civilians}    (7 players)

actual_win(p) =
    p's team won    →  1
    p's team lost   →  0
```

---

## 2. Base probability of a black win

Idea: the blacks and the reds are fundamentally unequal — by the very rules and mechanics of the game, not just by the number of players on a team. So the "fair" probability of a black win is not a constant 50% but an empirical value that can be tracked through the black team's win rate over the last N real games (for example, over the last 1000 games) — this way the asymmetry stays current rather than being fixed once and for all. This probability additionally depends on who was eliminated first in the game — an early loss of the sheriff or a game without a first killed player changes the typical balance of power.

**Current implementation** (one of the possible options):

```
p_black_win =
    array_avg(win_rate_array_no_fk),        if nobody was eliminated on the first night
    array_avg(win_rate_array_fk_sheriff),   if the sheriff was eliminated first
    array_avg(win_rate_array_fk_other),     in all other cases

p_red_win = 1 − p_black_win
```

Each of the three scenarios keeps its own sliding window of the last `N` valid games with the same first-elimination scenario — a new game is appended to this window, and the value of `p_black_win` for the next game of this scenario is taken as the average over the accumulated window. Each scenario has its own window size (section 8) — the common scenario keeps a wider window (stability over a large volume of data), the rare ones a narrower one (they reflect the real statistics faster and do not get stuck on an old sample). Until the window has been filled with real games, the value starts from a neutral value and is gradually replaced by real statistics as games of this scenario accumulate.

`p_black_win` is used only as the base component of the expectation (section 3) — the personal and team rating differences are added on top of it, not instead of it.

---

## 3. Win expectation: role, personal strength and team strength

Idea: the expectation of whether a player will win should take into account not only their personal rating relative to a particular opponent, but also how strong their whole team is relative to the opposing team — as well as the original role asymmetry from section 2. A logistic function is used, applied not to a base of 50% but to the empirical `p_black_win`/`p_red_win`.

**Current implementation** (one of the possible options):

```
win_expectation(baseline, ri, rj, team_i, team_j) =
    sigmoid( logit(baseline)
             + win.lambda_pair · ln(10) · (ri − rj) / win.rating_scale
             + win.lambda_team · ln(10) · (team_i − team_j) / win.rating_scale )
```

Here `baseline` is `p_black_win` if player i is black, otherwise `p_red_win` (section 2); `ri`/`rj` are the current personal "Win" ratings of the player and the opponent; `team_i`/`team_j` are the strength of one's own and the opposing team (section 4). `win.lambda_pair` and `win.lambda_team` are the weights with which the personal and team differences enter the expectation: the team component is the same for all of the player's comparisons with any opponent and reflects the overall balance of the game, while the personal one is only the player's own position within their team. `win.rating_scale` is the scale of the logistic expectation: how many points of rating difference are needed for a noticeable shift in expectation.

With `ri = rj` and `team_i = team_j` the expectation is exactly equal to `baseline` — that is, with full personal and team equality the only source of expectation left is the empirical role asymmetry.

---

## 4. Table context: team strength and volatility

This mechanism is about the CURRENT ratings of the players at the table, not about the outcome of a particular game — it answers the question "how sharply should each particular player's rating move", not "which team won".

### 4.1 Team strength and table strength

Idea: one strength characteristic is needed for each team — one's own and the opposing one — for the win expectation (section 3), and separately an overall strength characteristic of the whole table, which sets the reference point for personal volatility (4.2). Any suitable aggregate can be used for both.

**Current implementation** (one of the possible options):

```
own_team_min      = min( win(p) over own team )
opponent_team_min = min( win(p) over opposing team )
table_min         = min( win(p) over all 10 players )

own_team_strength      = own_team_min      + ( n_own_team      / Σ ( win(p) − own_team_min      + win.table_strength_eps )⁻³ ) ^ (1/3) − win.table_strength_eps
opponent_team_strength = opponent_team_min + ( n_opposing_team / Σ ( win(p) − opponent_team_min + win.table_strength_eps )⁻³ ) ^ (1/3) − win.table_strength_eps

table_win = table_min + ( n_players / Σ ( win(p) − table_min + win.table_strength_eps )⁻³ ) ^ (1/3) − win.table_strength_eps
```

All three values are the same power mean with exponent −3, applied to different groups of players: `own_team_strength`/`opponent_team_strength` take it within one team (3 or 7 players), `table_win` over all 10 at once. A power mean with a negative exponent is heavily biased towards the minimum values, so a team or a table is characterised primarily by how weak its weakest participant is, not by the typical/average level.

The formula is applied not directly to the ratings but to their DEVIATION from the minimum of the same group, with a constant `win.table_strength_eps` that brings the result back to the original scale. The reason: a power mean with a negative exponent applied directly to absolute rating values depends on where on the rating scale the whole group currently sits — a line-up that is the same in meaning (the same spread of strength between players) would give a DIFFERENT team or table strength, sometimes noticeably different, if all ratings were shifted by the same constant. Anchoring to the group minimum removes this dependence: the result does not change depending on where exactly on the absolute scale the whole group sits, only on how far apart the players are from each other.

`win.table_strength_eps` is not just a technical guard against division by zero (otherwise the weakest player's deviation from themselves would be exactly zero), but a full-fledged calibratable value: the smaller it is, the more harshly the formula reacts to the lagging player; the larger it is, the closer its behaviour is to an ordinary mean. At the time of writing it equals 900 — chosen to reproduce today's behaviour of the formula on real data as closely as possible (the difference from the previous, unadjusted formula is no more than 2–3 points on any of the real tables checked). This is a deliberate point for possible future calibration: if you want team/table strength to react more noticeably to a lagging player, `win.table_strength_eps` can be reduced — but at the moment the current value empirically looks normal and as expected.

### 4.2 Volatility relative to the table

Idea: a player's rating should move more gently when they are objectively stronger than the table — such a player has little left to prove, and they should not be shaken much in either direction. Important: volatility itself does not know the outcome of the game — it depends only on the strength difference BEFORE the game, not on who eventually won. A player gets a multiplier of the same size regardless of whether their team won or lost this particular game; what differs is not the multiplier but what it multiplies — the difference between the fact and the expectation from section 5. If a player who is weak relative to the table unexpectedly wins, this difference is large (the fact diverged strongly from the expectation); if they lose as expected, the difference is small (the loss was expected anyway), but in both cases the same volatility is used, known in advance from the balance of power: it amplifies or weakens what came out of the comparison with the expectation, rather than predicting the outcome itself.

**Current implementation** (one of the possible options):

```
m_win(p) = min( tanh_m( table_win − win(p), win.volatility_player.min, win.volatility_player.max, win.volatility_player.scale ), 1 )

tanh_m(diff, min, max, scale) =
    diff ≥ 0  →  1 + (max − 1) × tanh(diff / scale)
    diff < 0  →  1 − (1 − min) × tanh(|diff| / scale)
```

If the player's rating is BELOW the table strength (`diff > 0`), by the formula the multiplier would grow above 1, but the outer `min(..., 1)` cuts this off from above: a player who is weak relative to the table gets exactly the base multiplier of 1, not amplified. If the rating is ABOVE the table strength (`diff < 0`), the multiplier drops below 1, down to `win.volatility_player.min`: the rating of an already strong player changes more smoothly. So in practice this mechanism only protects strong players from sharp swings at weak tables; it does not increase the volatility of weak players beyond the base level.

---

## 5. Where everything meets: pairwise comparison with the opposing team

This is the core of the whole mechanism: the "Win" rating is built not on the fact of the win itself, out of context, but on how this fact relates to the expectation — separately for each opponent from the OTHER team. There is no comparison with players on one's own team: the outcome of the game is always the same for the player and for them, so a comparison with a teammate carries no information about the outcome.

```
win_sum(p) = Σ_q∈opposing_team  activity_factor(p, q) × min( m_win(p), m_win(q) ) × ( actual_win(p) − win_expectation(baseline, win(p), win(q), own_team_strength, opponent_team_strength) )

delta_win(p) = win.k × win_sum(p)
```

Here:

- **The sum runs only over players of the OTHER team** — for a black player these are the 7 red opponents, for a red player the 3 blacks. One's own team does not enter the sum at all.
- **`activity_factor(p, q)`** — a reducing multiplier: if p or q was eliminated first in the game, the term weighs less — the player eliminated first gives and gets less information about the game.
- **`min(m_win(p), m_win(q))`** — the SMALLER of the pair's two personal volatility multipliers is taken, not the multiplier of `p` itself. This is done deliberately and is critical for zero-sum (section 6).
- **`actual_win(p)`** — one and the same value (0 or 1) for all terms of one player's sum, since they have one outcome of the game; from term to term only `win_expectation` changes, since it is calculated separately for each particular pair.
- **`win.k`** — a single final scale of the delta per game.

---

## 6. Why the sum over the table is zero

This is not a coincidence but a direct consequence of how the formula is built: whatever one player gains from the comparison with a particular opponent from the other team, that opponent loses exactly the same amount (with the opposite sign) in the comparison with them.

Proof — for any pair `(i, j)` from different teams:

1. **`actual_win` is antisymmetric for a cross-team pair**: exactly one team wins, so if `actual_win(i) = 1`, then necessarily `actual_win(j) = 0` — and vice versa. The difference `actual_win(i) − actual_win(j)` is always `±1`, symmetric by the very nature of the game outcome.
2. **`win_expectation` is antisymmetric when the pair is swapped**: `win_expectation` for the pair `(j, i)` is exactly `1 − win_expectation` for the pair `(i, j)`. The reason is `logit(p_red_win) = logit(1 − p_black_win) = −logit(p_black_win)` (a standard property of the logit function), and both rating difference terms change sign when `i` and `j` are swapped — in total the whole argument of the sigmoid changes sign, which gives `sigmoid(−x) = 1 − sigmoid(x)`.
3. Hence `(actual_win(i) − win_expectation(i,j))` for the pair `(i, j)` is exactly minus the same expression for the pair `(j, i)`.
4. All the other multipliers — `activity_factor(i,j)` and `min(m_win(i), m_win(j))` — are the same in both directions: both are explicitly symmetric by construction and do not depend on whose point of view the calculation is done from.

In total: `raw(i,j) = −raw(j,i)` for every cross-team pair, and therefore the sum over all 10 players `Σ delta_win(p) = 0` — always, exactly, without approximations.

---

## 7. Hidden trend and visible rating

Idea: the raw delta for a game is an honest zero-sum signal, but on its own it can change too sharply from game to game. To smooth out such fluctuations without losing the signal itself, a split into two parallel numbers is introduced: a hidden trend, which accumulates the reaction to every game without smoothing, and a visible rating, which changes smoothly, being pulled towards the trend rather than jumping after it.

```
new_trend = trend + delta_win

gap = new_trend − win                                — gap between the trend and the visible rating
dir_match = sign(delta_win) == sign(gap)              — whether the direction of this game matches the direction of the gap

gap_component     = dir_match ? tanh(gap / win.smoothing.gap_scale) : 0
delta_r_component = tanh(delta_win / win.smoothing.delta_scale)

delta_visible = ( win.smoothing.trend_weight × gap_component + (1 − win.smoothing.trend_weight) × delta_r_component ) × win.smoothing.dv_max

new_win = win + delta_visible
```

- **`gap_component`** — "pulling towards the trend": the larger the accumulated gap, the more strongly the visible rating is pulled in its direction (saturating through `tanh`).
- **`dir_match`** — protection against "overcorrection": if this particular game pulls in the direction opposite to the already accumulated gap, the pull towards the trend is not applied at that moment — only the direct reaction to the game works.
- **`delta_r_component`** — the direct, unsmoothed reaction to the raw delta of THIS game.
- The final visible change is a weighted mix of these two components (`win.smoothing.trend_weight`), with a common ceiling of `win.smoothing.dv_max` points per game.

### 7.1 Trend bound (optional)

Since the trend accumulates with no limit on the step, over time it can drift very far from the visible rating (especially over a long streak of games in one direction). There is an optional parameter `win.trend_bound` (a number or disabled) that keeps the trend within `visible ± trend_bound`.

An important property of the implementation: the bound affects only what is actually stored in the trend — the whole calculation of the VISIBLE rating from section 7 above (gap, dir_match, gap_component, delta_r_component) is still done from the unbounded raw delta, as if there were no bound at all. The only difference is which number ends up being remembered as `trend` for the next game.

So that the bound does not break the zero-sum from section 6, it works in two passes:

1. For each player a coefficient `β` is found: by what percentage the raw delta would have to be cut so that the new trend does not go beyond `visible ± trend_bound`.
2. Each pairwise `raw(p, q)` from section 5 is multiplied not by `β(p)` alone but by `min(β(p), β(q))` — the same trick as with `min(m_win(p), m_win(q))`: a shared multiplier per pair rather than each player's personal multiplier, otherwise the sum over the table would no longer add up to zero.

---

## 8. Calibration: summary map of parameters

"Win" does not depend on the judging system and is calibrated with ONE shared set of values. The judge score plays no part in the calculation at all, so there is no need to calibrate it per judging system. The parameters in the table are given with their full config prefix (`win.`, `shared.`) — in the text above they are sometimes mentioned without it.

| Parameter | Step | Which mechanism it reproduces |
|---|---|---|
| `win.window_size.no_fk` / `win.window_size.fk_sheriff` / `win.window_size.fk_other` | 2 | Size of the sliding window for estimating `p_black_win`, one per first-elimination scenario |
| `win.lambda_pair` | 3 | Weight of the personal rating difference in the win expectation |
| `win.lambda_team` | 3 | Weight of the team strength difference in the win expectation |
| `win.rating_scale` | 3 | Scale of the logistic expectation — how many points of rating difference give a noticeable shift |
| `win.table_strength_eps` | 4.1 | How harshly team/table strength reacts to the weakest player (the smaller, the harsher); also brings the power mean of deviations back to the original rating scale |
| `win.volatility_player.min` / `win.volatility_player.max` / `win.volatility_player.scale` | 4.2 | How strongly personal strength relative to the table speeds up/slows down the movement of the rating |
| `shared.first_killed_factor` | 5 | Penalty for the lower informativeness of the game of the player eliminated first |
| `win.k` | 5 | Final shared scale of the delta per game |
| `win.smoothing.gap_scale`, `win.smoothing.delta_scale`, `win.smoothing.trend_weight`, `win.smoothing.dv_max` | 7 | Shape of mixing "pulling towards the trend" and "reaction to this game" into the visible rating |
| `win.trend_bound` | 7.1 | How far the trend is allowed to drift from the visible rating |

---

## 9. Summary chain (quick reference)

```
actual_win (fact: did the player's team win)
        │
        │              p_black_win / p_red_win (section 2)
        │                              │
        │                              ▼
        │                 win_expectation(baseline, personal rating, opponent rating,
        │                              own team strength, opposing team strength)
        │                              ▲
        │               ┌───────────────┴───────────────┐
        │        personal rating                team strength
        │        difference                      difference
        │                              │
        └──────────────┬───────────────┘
                        ▼
     for each opponent q from the OTHER team:
     actual_win vs win_expectation(q)
     raw(p,q) = activity_factor(p,q) × min(m_win(p), m_win(q)) × (actual_win(p) − win_expectation(q))
                        ▼
     win_sum(p) = Σ raw(p,q)  →  delta_win(p) = win.k × win_sum(p)
                        ▼
     new_trend = trend + delta_win   (accumulates without limit, or softly — trend_bound)
     gap, dir_match, gap_component, delta_r_component
                        ▼
     new_win = win + delta_visible
```

---

## 10. Caveats

Below are several deliberate features of the current implementation that are worth keeping in mind when interpreting the rating.

- **The rating does not take judge scores into account at all.** The only things that matter for it are whether the player's team won or lost (section 1), their current rating and the opponents' ratings. This is a deliberate feature, not an oversight: the "Win" rating measures exactly and only the team's result relative to the expectation, without trying to judge the quality of play within the game.
- **A win is a purely binary fact, with no notion of "confidence" or margin.** `actual_win ∈ {0, 1}` (section 1) does not distinguish a convincing win from a narrow one — only the bare fact that the team won or lost counts, not how decisive the advantage was. This follows directly from the nature of the game itself: a game of Mafia has no continuous scale of "how much one team outplayed the other" — there is only a binary outcome.
- **In pair volatility the more stable participant wins, not the better-informed one.** `min(m_win(p), m_win(q))` (section 5) takes the SMALLER of the pair's two personal volatilities. The multiplier of a player weaker than the table never rises above the base value of 1 — the outer `min(..., 1)` in section 4.2 cuts off the growth; the multiplier of a player stronger than the table, on the contrary, drops below 1 the more they outclass the table. Because of this the minimum of the pair almost always turns out to be the multiplier of the stronger participant in the comparison. This is a deliberate priority: the mechanism first and foremost protects strong players from sharp swings at weak tables, rather than speeding up the learning of players weaker than the table — if the priority were the other way round, section 5 would use `max` rather than `min`.
- **The sum over the opposing team is not normalised by its size.** A black player adds up terms over 7 red opponents, a red player over 3 blacks (section 5), and `delta_win = win.k × win_sum` uses the same `k` for both sides, without dividing by the number of terms. Since the terms of one game are strongly correlated in sign (rather than independent noise — they are all calculated from the same `actual_win` of this player), the blacks' sum is typically noticeably larger in absolute value than the reds': all else being equal, the rating of a don/mafia can move more strongly than that of a sheriff/civilian, simply because of the difference in team size. This can be explained by the fact that responsibility for the win or loss is shared among a different number of players depending on the team, and for the blacks the conditions are a little stricter: each of them is more responsible for their team's outcome than an individual red player. This has been left as is deliberately — it is not corrected by a separate multiplier or by dividing by the size of the opposing team.
- **Separate base probabilities per first-elimination scenario are a deliberate choice.** The three scenarios of section 2 (nobody eliminated on the first night, the sheriff eliminated first, someone else eliminated first) are taken into account not out of technical whim: both special cases — no early loss and an early loss of the sheriff specifically — happen practically at the very start of the game and significantly change how hard it is for one team or the other to win. That is exactly why `p_black_win` is split by scenario rather than being calculated as one shared probability for all games.
- **Counting the players' ratings twice — in the personal comparison and separately in team/table strength — is a deliberate decision.** The same rating of each of the 10 players takes part both in the personal difference (section 3) and in calculating team and table strength (section 4), so formally it is counted twice. This is done on purpose: in Mafia the overall strength of the table and of the team is not a secondary factor but something that largely determines whether a team can win at all, so it is worth taking into account separately from the personal comparison, even if both calculations rely on the same underlying ratings.
- **The choice of a power mean with exponent −3 is not arbitrary.** Both table strength and team strength (section 4.1) are calculated with a power mean heavily biased towards the minimum value, rather than an ordinary mean. The reason lies in the mechanics of the game itself: the weakest player at the table or on a team most often "drags" it down, and it is precisely that player whom the opposing team tries to play against in order to win. This is, of course, an abstract generalisation rather than a strict law, but on the whole it is a real tendency of the game, which a power mean with a negative exponent reflects better than an ordinary arithmetic mean.
- **The visible rating is not zero-sum, unlike the trend.** The proof in section 6 is strict for the raw delta `delta_win`: its sum over the table is exactly zero, without exceptions — and exactly the same is true for the per-game increment of the hidden trend (`new_trend = trend + delta_win`), since the trend accumulates this very raw delta directly. But what the user sees (`win_*`) is derived from it through smoothing (section 7) — `tanh` compression and the common ceiling `smoothing.dv_max`, non-linear transformations after which the sum of changes over the table no longer has to be zero. This is a deliberate decision: the goal of smoothing is to limit how much the visible rating can change in one game, not to preserve exact zero-sum at every step. The visible rating is protected from uncontrolled inflation or deflation not by an exact balance of sums but by two other mechanisms: pulling towards the trend (`gap_component`, section 7) and the limit on how far the trend is allowed to drift from the visible rating (`trend_bound`, section 7.1) — together they prevent the visible rating from diverging for long from the hidden trend, whose sum is honestly zero.
- **The whole mechanism reflects empirical experience, not mathematical necessity.** The choice of specific formulas, constants and priorities in this document is not the only possible one and is not dictated by mathematics as such — it is the result of observing real games and how they evolve over time, and of judgements about which things turned out to matter more than others. This does not mean the choice is arbitrary — behind every decision there is a specific reason, set out in the corresponding section — but none of them should be considered fixed once and for all. It is a subject of ongoing discussion and a point of growth, worth revisiting as new statistical and empirical observations accumulate.
