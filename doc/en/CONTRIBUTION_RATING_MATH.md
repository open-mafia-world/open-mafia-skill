# "Contribution" rating — mathematical core

The "Contribution" rating measures how well a player played in a particular game in the judge's opinion — regardless of whether their team won or lost. It is based not on the absolute judge score itself, but on how that score relates to the scores of the other nine players at the table and to what the player's current rating expected of them.

---

## 0. Idea: two different questions about the same score

Each player's judge score for a game is used by the mechanism TWICE, and these are two fundamentally different questions that do not affect each other:

1. **Trust in the judging of this game as a whole** — one shared number for the whole game: how finely and informatively the judge separated the 10 players by score. It is built from how the scores are grouped — who got the same score, how many such groups there are and how evenly the players are spread across them. This is called **resolution** (`resolution`).
2. **Who in a particular pair objectively played better, and how strongly this comparison should count** — the two raw judge scores are compared with each other directly: higher, lower or equal. Separately, for the same pair it is determined whether the comparison should be amplified if one of the two fell short of the judging system's norm while the other reached it.

Both questions start from the same raw `judge_score`, but after that they never cross: resolution has its own set of calibration parameters, the pair comparison has its own, and no parameter is shared between them. They meet only at the very end, in the pairwise comparison of players: resolution enters it as a shared multiplier for the whole game, and the outcome of the pair comparison as the thing that is directly compared with the rating-based expectation.

```
                         judge_score (raw score)
                                 │
                 ┌───────────────┴───────────────┐
                 │                                │
        BRANCH A: resolution             BRANCH B: pair comparison
     (trust in judging this game)        (who is higher/lower/equal)
                 │                                │
     groups of equal scores              outcome(score_i, score_j)
     Shannon entropy over groups          scale(score_i, score_j)
                 │                                │
            resolution ∈ [0, 1]          outcome ∈ {0, 0.5, 1}
                 │                                │
                 └──────────────┬─────────────────┘
                                 │
                    pairwise comparison (section 5):
              resolution as a shared multiplier for the game
         outcome(i,j) as what is compared with the expectation
```

Why it matters to understand it this way: a calibration parameter belonging to one question physically cannot affect the result of the other. The resolution reference does not change the outcome of a particular comparison, and the comparison amplification rule does not make the judging of this game "more reliable".

---

## 1. Input data

The input is the judge score (`value`) of each of the 10 players for this game:

```
judge_score(p) = round(value(p) × 10000) / 10000
```

Rounding to 4 decimal places is purely technical (protection against floating-point errors) and carries no meaning.

---

## 2. Branch A — resolution: trust in the judging of this particular game

Idea: different games give different quality of judging signal. If the judge clearly separated all 10 players by different scores, the signal is reliable. If almost everyone got the same score, the signal is not worth much trust, even if it formally lets you work out who is "higher" and who is "lower". Resolution is a number `resolution ∈ [0, 1]` that multiplies the entire further calculation and knows nothing about the players' current ratings — it is built only from the scores of this game.

### 2.1 Groups and entropy

- **Group** — the set of players with the same score. `score_groups` is how many distinct groups there are (from 1 if all scores match to 10 if all are different).
- **Shannon entropy** of the group distribution (taking group sizes into account, not just their number):

```
entropy = −Σ (|group| / 10) × ln(|group| / 10)      — over each separate group
entropy_norm = entropy / ln(10)                      — normalisation into [0, 1]
```

`entropy_norm = 0` if all 10 match (one group); `entropy_norm = 1` if all 10 are different (ten groups). Intermediate cases depend not only on the number of groups but also on how EVENLY the players are spread across them — "5 and 5" gives higher entropy than "9 in one group + 1 apart", although both cases have two groups.

### 2.2 Final resolution — two independent multipliers

```
resolution =
    score_groups ≤ 1  →  0
    otherwise          →  entropy_norm ^ entropy_power
                           × ( (score_groups − 1) / 9 ) ^ group_power
```

Each multiplier reproduces its own separate idea and is calibrated separately:

1. **`entropy_norm ^ entropy_power`** — how EVENLY the ties are distributed. The exponent `entropy_power` controls how strictly frequent ties are penalised: `< 1` "forgives" low entropy (raises it closer to 1 even with frequent ties), `> 1` does the opposite and is harsher.
2. **`((groups−1)/9) ^ group_power`** — purely the NUMBER of distinct groups, regardless of their sizes. The same exponent logic, but it looks at something else: you can have high entropy with a small number of groups (exactly "5 and 5"), and this multiplier separately penalises a small number of distinct values.

If the judge separated all 10 players by different scores (`score_groups = 10`, `entropy_norm = 1`), resolution equals 1 regardless of how close or far apart the numbers themselves are — the spread of the scores does not enter this decision.

---

## 3. Branch B — pair comparison: outcome and amplification

Idea: a raw score on its own says nothing about how good or bad the game was in an absolute sense — but for TWO players at the same table in the same game, their scores are already fully comparable with each other, directly, without any conversion to another scale.

### 3.1 Comparison outcome

```
outcome(score_i, score_j) =
    score_i >  score_j  →  1     (i played better)
    score_i == score_j  →  0.5   (tie — same judge score)
    score_i <  score_j  →  0     (i played worse)
```

The two raw judge scores of this game are compared directly. A tie is handled as in a typical logistic rating system: `outcome = 0.5` means the actual result matched the middle of the expected scale; if the pair's ratings differ, the player with the higher rating loses slightly on a tie and the weaker one gains slightly.

### 3.2 Pair scale function

Idea: the comparison needs sensitivity to the scale of the particular judging system — what looks extreme in one system may be normal in another. Any function of the pair's scores can be used for this; at the moment a simple and universal one has been chosen, based only on the notion of a "normal game" — it requires no calibration beyond the already existing norm boundary.

**Current implementation** (one of the possible options):

```
scale(score_i, score_j) = contribution_pair_scale(lo, hi, normal_game_boundary)
    where lo = min(score_i, score_j), hi = max(score_i, score_j)

contribution_pair_scale(lo, hi, boundary) =
    lo < boundary && hi >= boundary  →  2
    otherwise                         →  1
```

The function always receives an already sorted pair of scores (lower, higher), so it is automatically symmetric: `scale(i,j)` and `scale(j,i)` always match — this is guaranteed by the way it is called, not by a special symmetric form of the formula itself.

The comparison coefficient is doubled exactly when one of the pair played BELOW the norm (`normal_game_boundary`) and the other did not play below the norm. If both are below the norm, or both are not below it, the coefficient is 1. Meaning: whoever played worse than the norm is penalised more strongly specifically in comparison with someone who did not break the norm.

`normal_game_boundary` is used only here (as the threshold for this rule) and separately in section 5 of [COMBINE_RATING_MATH.md](COMBINE_RATING_MATH.md) — for the winner/loser "floor/ceiling" protection; otherwise it does not affect the strength of comparing players with each other.

This is the main extension point of the mechanism for a particular federation/organisation: if it wants to interpret its own results more precisely, this is done here, by overriding `contribution_pair_scale`, without changing the rest of the maths.

---

## 4. Table context: table strength and player volatility

This is a third, independent mechanism — it is not about the scores of this game but about the CURRENT ratings of the players at the table, and it answers the question "how sharply should each particular player's rating move", not "who played better".

### 4.1 Table strength

Idea: one "table strength" characteristic based on the current ratings of all 10 players is needed — it sets the reference point for the volatility multiplier (4.2). Any suitable aggregate can be used for this.

**Current implementation** (one of the possible options):

```
contribution_min = min( contribution(p) )     — minimum over all 10 players at the table

table_contribution = contribution_min + ( n_players / Σ ( contribution(p) − contribution_min + contribution.table_strength_eps )⁻³ ) ^ (1/3) − contribution.table_strength_eps
```

A power mean with exponent −3 is not an ordinary mean but a mean heavily pulled towards the weakest players at the table: "table strength" is determined primarily by how weak its WEAKEST participant is, not by a simple average of everyone.

The formula is applied not directly to the ratings but to their DEVIATION from the table minimum (`contribution(p) − contribution_min`), with a constant `contribution.table_strength_eps` that brings the result back to the original scale. The reason: a power mean with a negative exponent applied directly to absolute rating values depends on where on the rating scale the whole group currently sits — a table that is the same in meaning (the same spread of strength between players) would give a DIFFERENT table strength, sometimes noticeably different, if all ratings were shifted by the same constant. Anchoring to the table minimum removes this dependence: the result does not change depending on where exactly on the absolute scale the whole group sits, only on how far apart the players are from each other.

`contribution.table_strength_eps` is not just a technical guard against division by zero (otherwise the weakest player's deviation from themselves would be exactly zero), but a full-fledged calibratable value: the smaller it is, the more harshly the formula reacts to the lagging player; the larger it is, the closer its behaviour is to an ordinary mean. At the time of writing it equals 900 — chosen to reproduce today's behaviour of the formula on real data as closely as possible (the difference from the previous, unadjusted formula is no more than 2–3 points on any of the real tables checked). This is a deliberate point for possible future calibration: if you want "table strength" to react more noticeably to a lagging player, `contribution.table_strength_eps` can be reduced — but at the moment the current value empirically looks normal and as expected.

### 4.2 Player volatility (m_contribution)

Idea: a player who is weaker than their table potentially gains more experience from this game — new information about themselves relative to stronger opponents — so their rating should be more volatile in order to reflect it faster. And if a player who is objectively much stronger than the table plays at it, their rating should be less strongly affected by this particular game. Overall this serves two purposes: so that strong players are not afraid to take part in games at weaker tables (without risking a big rating jump in either direction), and to prevent rating farming by playing at obviously weak tables. Any shape of this dependence can be chosen; a saturating hyperbolic tangent is currently used.

**Current implementation** (one of the possible options):

```
m_contribution(p) = tanh_m( table_contribution − contribution(p), volatility.min, volatility.max, volatility.scale )

tanh_m(diff, min, max, scale) =
    diff ≥ 0  →  1 + (max − 1) × tanh(diff / scale)
    diff < 0  →  1 − (1 − min) × tanh(|diff| / scale)
```

If the player's rating is BELOW the table strength (`diff > 0`), the multiplier grows above 1, up to `volatility.max`: such a player's rating should move more sharply. If the rating is ABOVE the table strength, the multiplier drops below 1, down to `volatility.min`: the rating of an already established strong player changes more smoothly.

`volatility.min`, `volatility.max`, `volatility.scale` are shared constants for all judging systems (not calibrated per model/period): this mechanism reproduces the engine-wide property "underrated ratings move faster than overrated ones", not something specific to a particular federation.

---

## 5. Where the branches meet: pairwise comparison of players

This is the core of the whole mechanism: the contribution rating is built not on the ABSOLUTE judge score, but on HOW one player played relative to each of the other 9 at the table, comparing this with what the current rating expected. Here branch A (resolution) and branch B (pair outcome and scale) meet for the first time — but in different ways: resolution enters as a shared multiplier for the whole game, the comparison outcome as the value actually being compared.

### 5.1 Who should have played better — by rating (logistic expectation)

Idea: a function is needed that predicts the expected outcome of the pair's comparison from their ratings — in general, this core can be ANY function that takes the pair's ratings as input and returns such an expectation. The only property required by the rest of the mechanism is antisymmetry when the pair is swapped (section 6); everything else about the shape of this function is replaceable.

**Current implementation** (one of the possible options) — the classic logistic expectation:

```
contribution_expectation(ra, rb) = sigmoid( logit(baseline) + ln(10) × (ra − rb) / rating_scale )
```

The only difference is that "winning" here means "a higher judge score". `baseline = 0.5` is the expectation for EQUAL ratings; it is fixed at exactly 0.5 and is NOT calibrated per system: this is a necessary condition for zero-sum (section 6). `rating_scale` is also a shared constant for all systems: how many points of rating difference are needed for a noticeable shift in expectation.

### 5.2 Assembling the raw signal of one pair

For player `p` and each of the 9 opponents `q`:

```
raw(p, q) = contribution_gain × resolution × activity_factor(p, q)
            × scale(score(p), score(q))
            × min( m_contribution(p), m_contribution(q) )
            × ( outcome(score(p), score(q)) − contribution_expectation(contribution(p), contribution(q)) )
```

Both branches and the table context meet here:
- **`resolution`** — all of branch A, one number for the whole game, the same for every pair at this table.
- **`scale(score(p), score(q))`** and **`outcome(score(p), score(q))`** — all of branch B: `scale` amplifies the comparison or not, `outcome` is what is actually compared with the expectation.
- **`contribution_gain`** — the SCALE of the raw signal, calibrated per judging system: different systems systematically give signals of different size, and this parameter brings them to a common scale without changing what counts as a "good" or "bad" game (that is done by `normal_game_boundary` from branch B, not by this parameter).
- **`activity_factor(p, q)`** — a reducing multiplier if p or q had `is_first_killed = 1`: the player eliminated first gets less information about the game, and the game gives less information about them. A shared constant (`first_killed_factor`), not calibrated per judging system.
- **`min(m(p), m(q))`** — the table context from section 4: the SMALLER of the two personal volatility multipliers is taken, not the multiplier of `p` itself. This is done deliberately and is critical for zero-sum (section 6).

### 5.3 Sum over the table and delta for the game

```
contribution_sum(p) = Σ raw(p, q)     over all 9 opponents q
delta_contribution(p) = k × contribution_sum(p)
```

`k` is a single final scale for all systems (unlike `contribution_gain`, which is calibrated individually at the start of the chain). `delta_contribution(p)` is the "raw", not yet smoothed change in player p's contribution rating for this game.

---

## 6. Why the sum over the table is zero

This is not a coincidence but a direct consequence of how the formula is built — and that is exactly why the engine calls contribution **zero-sum per table**: whatever one player gains from the comparison with a particular opponent, that opponent loses exactly the same amount (with the opposite sign) in the comparison with them.

Proof — for any pair `(i, j)`:

1. **`outcome` is antisymmetric**: `outcome(j,i) = 1 − outcome(i,j)` for any two scores (1↔0 for different scores, 0.5↔0.5 for a tie — `1 − 0.5 = 0.5`, so the property holds here too).
2. **`contribution_expectation` is antisymmetric exactly when `baseline = 0.5`**: `logit(0.5) = 0`, so `contribution_expectation(rb, ra) = 1 − contribution_expectation(ra, rb)`. This is precisely why `baseline` is fixed at 0.5 rather than being a free calibration parameter like the others.
3. Hence `(outcome − expectation)` for the pair `(j,i)` is exactly minus the same expression for `(i,j)`.
4. All the other multipliers — resolution, `contribution_gain`, `activity_factor`, `scale(score(p), score(q))`, `min(m_i,m_j)` — **are the same in both directions**. For `scale` this is not a coincidence but a direct consequence of it always being called with an already sorted pair of scores (section 3.2) — so `scale(i,j)` and `scale(j,i)` are literally the same call with the same arguments.

In total: `raw(i,j) = −raw(j,i)` for every pair, and therefore the sum over all 10 players `Σ delta_contribution(p) = 0` — always, exactly, without approximations.

---

## 7. Hidden trend and visible rating

Idea: the rating calculated above is an honest zero-sum signal, but on its own it can change too sharply from game to game, including because of one atypical game or a whole streak of bad/good games in a row. To make this movement more stable — to smooth out such fluctuations without losing the signal itself — a split into two parallel numbers is introduced: a hidden **trend**, which accumulates the reaction to every game without smoothing, and a visible **rating**, which changes smoothly, being pulled towards the trend rather than jumping after it.

In practice it looks like this: the hidden trend accumulates raw deltas with no limit on the step, and the visible rating changes smoothly, "pulled" towards the trend. All constants of this step are shared for all judging systems; no calibration is provided here.

```
new_trend = trend + delta_contribution

gap = new_trend − contribution                      — gap between the trend and the visible rating
dir_match = sign(delta_contribution) == sign(gap)    — whether the direction of this game matches the direction of the gap

gap_component    = dir_match ? tanh(gap / gap_scale) : 0
delta_r_component = tanh(delta_contribution / delta_scale)

delta_visible = ( trend_weight × gap_component + (1 − trend_weight) × delta_r_component ) × dv_max

new_contribution = contribution + delta_visible
```

- **`gap_component`** — "pulling towards the trend": the larger the accumulated gap, the more strongly the visible rating is pulled in its direction (saturating through `tanh`).
- **`dir_match`** — protection against "overcorrection": if this particular game pulls in the direction OPPOSITE to the already accumulated gap (the player is turning around), the pull towards the trend may not be applied at that moment — only the direct reaction to the game works. Following this gate can be enabled or disabled — this is configured depending on what fits the spirit of the particular rating system: how strictly it should protect against "overcorrection" against a recent turnaround.
- **`delta_r_component`** — the direct, unsmoothed reaction to the raw delta of THIS game.
- The final visible change is a weighted mix of these two components (`trend_weight`), with a common ceiling of `dv_max` points per game.

### 7.1 Trend bound (optional)

Since the trend accumulates with no limit on the step, over time it can drift very far from the visible rating (especially over a long streak of games in one direction). There is an optional parameter `trend_bound` (a number or disabled) that keeps the trend within `visible ± trend_bound`.

An important property of the implementation: the bound affects **only** what is actually stored in the trend — the whole calculation of the VISIBLE rating from section 7 above (gap, dir_match, gap_component, delta_r_component) is still done from the UNBOUNDED raw delta, as if there were no bound at all. The only difference is which number ends up being remembered as `trend` for the next game.

So that the bound does not break the zero-sum from section 6, it works in two passes:
1. For each player a coefficient `β` is found: by what percentage the raw delta would have to be cut so that the new trend does not go beyond `visible ± trend_bound`.
2. Each pairwise `raw(p, q)` from section 5.2 is multiplied not by `β(p)` alone but by `min(β(p), β(q))` — the same trick as with `min(m_i, m_j)`: a shared multiplier per pair rather than each player's personal multiplier, otherwise the sum over the table would no longer add up to zero. Because of this, when several players at the same table are clamped at once, the `±trend_bound` boundary is not held absolutely exactly (a small "overshoot" is possible), but zero-sum is held exactly, always.

---

## 8. Calibration: summary map of parameters

Different federations use different judging systems (`iMafia`, `FIIM`, `EG`, `CLM`, `MafGame`, …), and even one system can change its methodology over time — each has its own set of the values below, with its own date range. The remaining parameters are shared engine constants, the same for all systems and periods. In table 8.2 the parameters are given with their full config prefix (`contribution.`, `shared.`) — in the text above they are often mentioned without it for brevity.

### 8.1 Calibrated per judging system and period

| Parameter | Step | Which mechanism it reproduces |
|---|---|---|
| `entropy_power` | A — resolution | How strictly to penalise uneven distribution of tied scores |
| `group_power` | A — resolution | How strictly to penalise a small number of distinct scores |
| `normal_game_boundary` | B — pair scale (3.2) | The "normal" game threshold: determines whether the ×2 amplification fires in the pair comparison |
| `contribution_gain` | Signal assembly (5.2) | The scale of this system's raw signal (evens out the systematic difference in signal size between systems) |

### 8.2 Shared engine constants (not calibrated per system)

| Parameter | Step | Which mechanism it reproduces |
|---|---|---|
| `contribution.baseline` | 5.1 | Expectation for equal ratings; fixed at 0.5 for the sake of zero-sum |
| `contribution.rating_scale` | 5.1 | Scale of the logistic expectation — how many points of rating difference give a noticeable shift |
| `shared.first_killed_factor` | 5.2 | Penalty for the lower informativeness of the game of the player eliminated first |
| `contribution.table_strength_eps` | 4.1 | How harshly table strength reacts to the weakest player (the smaller, the harsher); also brings the power mean of deviations back to the original rating scale |
| `contribution.volatility.min` / `contribution.volatility.max` / `contribution.volatility.scale` | 4.2 | How strongly a player's personal volatility speeds up/slows down the movement of their rating |
| `contribution.k` | 5.3 | Final shared scale of the delta per game |
| `contribution.smoothing.gap_scale`, `contribution.smoothing.delta_scale`, `contribution.smoothing.trend_weight`, `contribution.smoothing.dv_max` | 7 — smoothing | Shape of mixing "pulling towards the trend" and "reaction to this game" into the visible rating |
| `contribution.trend_bound` | 7.1 | How far the trend is allowed to drift from the visible rating |

### 8.3 Pair scale function — separate from the calibration tables

`contribution_pair_scale` (section 3.2) is not a numeric parameter but a formula, the same for all judging systems and periods: the rule "×2 when the pair straddles the norm, otherwise ×1" is currently not calibrated individually per system. It is defined as a separate, independently editable function precisely so that, if necessary, it can be rewritten into something else — for example, something depending on the particular system or on the size of the gap between the scores, not only on which side of the norm each member of the pair ended up on.

---

## 9. Summary chain (quick reference)

```
judge_score (raw score of each of the 10 players)
        │
        ├──────────────────────────┐
        ▼                          ▼
  BRANCH A: resolution       BRANCH B: pair comparison
  score_groups,               outcome(score_i, score_j)
  entropy_norm                scale(score_i, score_j, normal_game_boundary)
        │                          │
   resolution ∈ [0,1]         outcome ∈ {0, 0.5, 1}
        │                          │
        └─────────────┬────────────┘
                       ▼
     for each pair (p, q):
     outcome(score) vs contribution_expectation(rating)
     raw(p,q) = contribution_gain × resolution × activity_factor × scale
                × min(m_contribution(p), m_contribution(q)) × (outcome − expectation)
                       ▼
     contribution_sum(p) = Σ raw(p,q)  →  delta_contribution(p) = k × contribution_sum(p)
                       ▼
     new_trend = trend + delta_contribution   (accumulates without limit, or is softly bounded by trend_bound)
     gap, dir_match, gap_component, delta_r_component
                       ▼
     new_contribution = contribution + delta_visible
```

---

## 10. Trust in the judging system as a whole

Idea: this whole calculation only makes sense when the scores of a particular judging system are a meaningful signal rather than arbitrary numbers. This requires a certain amount of trust: in the scoring methodology itself and in the federation/organisation that applies and maintains it. If there is no such trust, or the methodology changes too often and chaotically, there is no point in using its scores to calculate the contribution rating — it is better to explicitly opt out of this calculation for such a system (entirely or for a specific period) than to calculate from unreliable data.

---

## 11. Caveats

Below are several deliberate limitations of the current implementation that are worth keeping in mind when interpreting the rating.

- **Insensitivity to the size of the score.** Branch B sees only the order of the scores (section 3.1: `outcome` — strictly higher, lower or equal) and which side of `normal_game_boundary` each member of the pair is on (section 3.2). The size of the gap between scores does not enter the calculation: all else being equal, the rating change for a score of −1.6 will be the same as for −0.6, and for +0.8 the same as for +1.4. This is a deliberate choice: the participating judging systems use stable and meaningful score steps, and this precision is sufficient for now. Whether a particular federation needs finer sensitivity to the size of the gap is a question the players themselves should raise — by asking the federation to clarify how this should work in the current version of its judging scale. That is exactly what the extension point `contribution_pair_scale` (section 3.2) exists for in the mechanism.
- **The ×2/×1 rule is a deliberately simple baseline, not a limit.** `contribution_pair_scale` was introduced as the simplest option that distinguishes only two situations: someone in the pair fell short of the norm, or both are in the same category. It makes no claim to precision; it is a starting extension point for the future — the same thing the previous point talks about.
- **In pair volatility the more stable participant wins, not the better-informed one.** `min(m_contribution(p), m_contribution(q))` (section 5.2) takes the SMALLER of the two personal volatilities. Since the volatility of a player weaker than the table grows above 1, while that of a player stronger than the table drops towards `volatility.min` (section 4.2), the minimum of the pair almost always turns out to be the volatility of the stronger participant in the comparison. This is a deliberate priority: the mechanism was created first and foremost to protect strong players from sharp jumps at weak tables, not to speed up the learning of weaker ones — if the priority were the other way round, section 5.2 would use `max` rather than `min`.
- **The visible rating is not zero-sum, unlike the trend.** The proof in section 6 is strict for the raw delta `delta_contribution`: its sum over the table is exactly zero, without exceptions — and exactly the same is true for the per-game increment of the hidden trend (`new_trend = trend + delta_contribution`), since the trend accumulates this very raw delta directly. But what the user sees (`contribution_*`) is derived from it through smoothing (section 7) — `tanh` compression and the common ceiling `dv_max`, non-linear transformations after which the sum of changes over the table no longer has to be zero. This is a deliberate decision: the goal of smoothing is to limit how much the visible rating can change in one game, not to preserve exact zero-sum at every step. The visible rating is protected from uncontrolled inflation or deflation not by an exact balance of sums but by two other mechanisms: pulling towards the trend (`gap_component`, section 7) and the limit on how far the trend is allowed to drift from the visible rating (`trend_bound`, section 7.1) — together they prevent the visible rating from diverging for long from the hidden trend, whose sum is honestly zero.
- **The whole mechanism reflects empirical experience, not mathematical necessity.** The choice of specific formulas, constants and priorities in this document is not the only possible one and is not dictated by mathematics as such — it is the result of observing real games and how they evolve over time, and of judgements about which things turned out to matter more than others. This does not mean the choice is arbitrary — behind every decision there is a specific reason, set out in the corresponding section — but none of them should be considered fixed once and for all. It is a subject of ongoing discussion and a point of growth, worth revisiting as new statistical and empirical observations accumulate.
