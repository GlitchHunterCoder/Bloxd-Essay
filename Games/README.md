Below is an essay which is still being edited detailed the custom game algorithms working and how to make games naturally optimised towards this goal
```diff
# CUSTOM GAME ALGORITHM
# Study

+ ## Our Data
+   - ### 5 Variables
+   - ### Formula
+   - ### Hints

+ ## The Problem
+   - Hidden `engagementScore`
+   - Ranking-only information
+   - Turning ranking into inequalities

+ ## The Solution
+   - ### Formula Space
+   - ### THE MOMENT YOU HAVE BEEN WAITING FOR
+   - Final formula

+ ## METHOD
+   - ### GAINS
+   - ### DRAINS
+   - ### Disclaimer

  ## MODELS
    - ### PLAYER BEHAVIOR
-   - ### STATISTIC EFFECTS
-   - ### GAME LOOPS

- ## APPLICATION
-   - ### DESIGNING A GAME
-     - Start from player behavior
-     - Map behavior → statistics
-     - Map statistics → score effect
-     - Build favorable loops
-   - ### OPTIMIZATION
-     - What to prioritize
-     - What to avoid
-     - Optimize ratios rather than blindly maximizing numbers
-   - ### EXAMPLES
-     - Good vs bad game loops
-     - Different designs producing different scores
-     - Practical examples

- ## VERIFICATION
-   - ### Ranking Accuracy
-     - Test against all 796 records
-     - 100% pairwise ordering accuracy
-     - Kendall tau = 1
-     - Spearman rho = 1
-   - ### Exponent Validation
-     - Compare values around `0.75`
-     - Show why `0.75` is significant
-   - ### Equivalent Forms
-     - Derived-metric form
-     - Ratio form
-     - Raw-variable form

- ## LIMITATIONS
-   - ### What We Actually Proved
-   - ### What We Did Not Prove
-   - ### Dataset Limitations

- ## CONCLUSION
-   - ### The Formula
-   - ### What It Rewards
-   - ### What It Discourages
-   - ### Core Design Principle
-   - ### What This Means For Developers

- ## APPENDIX
-   - ### Dataset
-   - ### Formula Code
-   - ### Mathematical Derivation
-   - ### Search Method
-   - ### Test Results
```
# CUSTOM GAME ALGORITHM
# Study
## Our Data
### 5 Variables
```js
"engagementScore_plays": 452028, // = P
"engagementScore_activeDays": 7, // = D
"engagementScore_uniquePlayers": 217590, // = U
"engagementScore_totalPlaytimeMs": 378183683010, // T
"engagementScore_playtimeSamples": 451147 // S
```
- Update, can be found here in 2 places now not just 1
comes together to create 6th metric which we cant see `engagementScore`
### Formula
```js
P^a * D^b * U^c * T^d * S^e

where P - S our engagementScore points,
and a - e exponents

we can assume a - e are rather small magatudes, eg -2 to 2
```
### Hints
This Formula computes some more metrics, and are a hint to what makes them
- Found in Src, edited for readability
```js
function more(data) {
    const {engagementScore_plays: P, engagementScore_activeDays: D, engagementScore_uniquePlayers: U, engagementScore_totalPlaytimeMs: T, engagementScore_playtimeSamples: S} = data;
    return {
        playsPerDay: void 0 !== P && D ? P / D : null, // -> P / D = plays / activeDays // 
        avgPlaytimeMinutes: void 0 !== T && S ? T / S / 6e4 : null, // -> T / S / 6e4 = totalPlaytimeMs / playtimeSamples / 60,000
        retention: void 0 !== P && U ? P / U : null // -> P / U  = plays / uniquePlayers
    }
}
```
- you can also find these by going into `settings -> general -> show_debug_overlay` and toggling it on
## The Problem
### Cant see it 
- only way to tell rankings, is based off these
- the way we find these is by by thinking of the formula as F(5 argsA) > F(5 ArgsB)
- and creating a space which we can work with, aka we can ask which is bigger than which, but not by how much
### Rankings Only
```js
"rankCategories": [
    {
        "rankCategory": "playerSchematic",
        "rank": 1
    },
    {
        "rankCategory": "Minigames",
        "rank": 1
    }
]
```
Is all we can see, and from that it isnt much so we have to be pretty inventive as to how we solve that, and as youll see we took some wild approach
## The Solution
### Formula Space
```js
E = P^a * D^b * U^c * T^d * S^e
//take logarithms
log(E) = a * log(P) + b * log(D) + c * log(U) + d * log(T) + e * log(S)

//now we got this
```
now comparing 2 is as follows
```js
E1 > E2
// means
a * log(P1) + b * log(D1) + c * log(U1) + d * log(T1) + e * log(S1) 
>
a * log(P2) + b * log(D2) + c * log(U2) + d * log(T2) + e * log(S2)
// simplify by subtracting all from right
a * log(P1) + b * log(D1) + c * log(U1) + d * log(T1) + e * log(S1) - (a * log(P2) + b * log(D2) + c * log(U2) + d * log(T2) + e * log(S2)) > 0
a * log(P1) + b * log(D1) + c * log(U1) + d * log(T1) + e * log(S1) - a * log(P2) - b * log(D2) - c * log(U2) - d * log(T2) - e * log(S2) > 0
a * (log(P1)-log(P2)) + b * (log(D1)-log(D2)) + c * (log(U1)-log(U2)) + d * (log(T1)-log(T2)) + e * (log(S1)-log(S2)) > 0
```
note how this lot has now become an inequality in 5 dimensions
but our other metrics come to help us out
```js
playsPerDay = P * D^-1
avgPlaytimeMinutes = T * S^-1
retention = P * U^-1
// so if we assume its actually

E = (playsPerDay)^x * (avgPlaytimeMinutes)^y * (retention)^z
//then we get to the following by simplifying
E = (P * D^-1)^x * (T * S^-1)^y * (P * U^-1)^z

E = P^x * D^-x * T^y * S^-y * P^z * U^-z

E = P^(x+z) * D^-x * T^y * S^-y * U^-z
//and rearrange
E = P^(x+z) * D^-x * U^-z * T^y * S^-y
//but wait, didnt this look familiar, say hello to our previous expression
E = P^a * D^b * U^c * T^d * S^e

//that means
a = x+z
b = -x
c = -z
d = y
e = −y

//and we can then make these restrictions

a + b + c = 0
d + e = 0
```
so thats the theory, in pratice, we just batched the 3 exponent version to see what worked, now note there are mathematical tests we can use to verify that those specific exponents are the case, but ill save those for a later chapter and so some time later

### THE MOMENT YOU HAVE BEEN WAITING FOR
```js
// FINAL FORMULA
E = playsPerDay ^ 0.75 * avgPlaytimeMinutes * retention
// or converted to our base units

E = (P * D^-1)^0.75 * (T * S^-1)^1 * (P * U^-1)^1

E = P^1.75 * D^-0.75 * T^1 * S^-1 * U^-1
```
```js
function engagementScore(x) {
    const playsPerDay =
        x.engagementScore_plays /
        x.engagementScore_activeDays;

    const avgPlaytimeMinutes =
        x.engagementScore_totalPlaytimeMs /
        x.engagementScore_playtimeSamples /
        60000;

    const retention =
        x.engagementScore_plays /
        x.engagementScore_uniquePlayers;

    return (
        playsPerDay ** 0.75 *
        avgPlaytimeMinutes *
        retention
    )
}
```
```js
engagementScore
  = plays^1.75 * 
    activeDays^-0.75 * 
    totalPlaytimeMs^1 * 
    playtimeSamples^-1
    uniquePlayers^-1

  = playsPerDay ** 0.75 *
    avgPlaytimeMinutes *
    retention
```
now these results may look odd, why are you being punished for activeDays, uniquePlayers
but think about it like so, if all other variables stay constant, but 1 changes, thats the effect it has
- eg game with short arrival and quick gain in plays, is better than a game which was slower to get to those plays
similar reasoning can be followed for the rest, so the way to get your game on leaderboard QUICKLY is as follows
## METHOD

### GAINS
> - These are the variables you generally want to increase 
> - However, increasing a gain alongside its corresponding drain does not necessarily increase your score - the important thing is how they change relative to each other
### Plays — Biggest Driver

- Keep `plays` high This is your driving force: the higher it goes, the more it pushes your chances of getting onto the big board
- This drives BOTH:
  - `retention` — ×1 exponent
  - `playsPerDay` — ×0.75 exponent
- Combined, this gives `plays` a **×1.75 exponent** No wonder it is the most valuable stat

So, assuming all other stats don't change:

- **×6 increase in plays → ×23 increase in score**

> **Increase**

Design for getting many plays:

- Make the game immediately understandable
- Make the thumbnail enticing so people want to join **Don't rely on clickbait, though — misleading clickbait is punished**
- Build gameplay that entices players to bring others in
- Make sure those players actually play through the game rather than immediately leaving

- **You should care about this the MOST**

### totalPlaytimeMs
- Keep `totalPlaytimeMs` high It is not as powerful as `plays`, but it is still very important
- This drives `avgPlaytimeMinutes` — ×1 exponent

So, assuming all other stats stay constant:

- **×2 increase in total playtime → ×2 increase in score**
- This assumes `playtimeSamples` does not increase proportionally

> **Increase**

Design around sustained sessions, not just getting someone to join once:

- Give players a goal to grind towards
- Give incentives for staying
- Give players reasons to stay even after the main goal ends
### DRAINS

> - These aren't inherently bad stats They become a drain when they increase without the corresponding gain increasing enough to cover them
> - If the gain rises alongside the drain, the penalty can be reduced or completely cancelled out

### activeDays

- `activeDays` can hurt your score long term if your players are not generating enough additional plays as they return
- This drains `playsPerDay` — ×0.75 exponent

So, assuming all other stats stay constant:

- **×4 increase in activeDays → ×3 decrease in score**

> **Minimize relative to plays**

The goal isn't necessarily to have fewer active days The goal is to make sure returning players generate enough additional plays to compensate for the increase

### playtimeSamples

- This is one of the biggest drains
- Every additional play session increases `playtimeSamples`, which lowers your average playtime if total playtime doesn't increase by the same amount
- This drains `avgPlaytimeMinutes` — ×1 exponent

So, assuming all other stats stay constant:

- **×2 increase in playtimeSamples → ×2 decrease in score**

> **Minimize relative to totalPlaytime**

Design for longer sessions rather than lots of tiny sessions:

- Avoid gameplay loops that constantly make players leave and rejoin
- Don't optimize for repeated short sessions
- Try to turn short sessions into longer, meaningful ones
- **Long sessions > many tiny sessions**

### uniquePlayers — The Brainrot Killer

- `uniquePlayers` can hurt your score when those new players don't generate additional plays
- This drains `retention` — ×1 exponent
- **This is the brainrot killer:** attracting players with short attention spans can tank the engagement score if they don't come back or generate additional plays

So, assuming all other stats stay constant:

- **×2 increase in uniquePlayers → ×2 decrease in score**

> **Minimize relative to plays**

You don't want players to simply try the game once
You want them to play, stay, come back, and play again
## Models
### Players

Up until now, I have been making an overarching assumption:
that each of these stats rise and fall independently.

However, this is often not the case.

A single player action can affect multiple statistics at once.
Likewise, increasing one statistic can naturally cause another
statistic to increase alongside it.

Understanding these relationships is important because the formula
does not simply reward or punish individual statistics. It rewards
the relationships between them.

> Stat to Player

- **Joins**
  - +1 Plays

- **Play for a minute**
  - +1 totalPlaytimeMs

- **Leave**
  - +1 playtimeSamples

- **1 day passes**
  - +1 activeDays

- **Play the game again**
  - +1 Plays
  - +1 playtimeSamples
  - + totalPlaytimeMs

- **A unique player plays the game**
  - +1 uniquePlayers
  - +1 Plays
  - +1 playtimeSamples
  - + totalPlaytimeMs
> Player to Stats

Now for this, we can also factor in something we didnt add in before, 1 stat decreasing without a relevent increase often doesnt happen
So below we will also factor in what other stats naturally change
We will assume that a games stats will either increase linearly, `Plays`, or not increase as based on game itself, `avgPlaytimeMinutes`

- **Plays**
  - Means the total amount of plays which the game has gotten.
  - More/less plays of your game.
  - This should never be decreased; this stat has no benefit from being lower.

- **totalPlaytimeMs**
  - Means more total hours across all players being put into your game.
  - More/less total hours.
  - This is another stat which has no benefit from being lower.

- **activeDays**
  - Means the number of distinct days on which the game has been played.
  - More/less active days.
  - More active days can be useful for bringing players back, but
    the formula does not directly reward the number itself.
  - It becomes a drain when plays do not increase enough alongside it.

- **playtimeSamples**
  - Means the number of separate play sessions contributing to
    totalPlaytimeMs.
  - More/less sessions.
  - Increasing this without increasing totalPlaytimeMs by enough
    lowers average playtime.
  - Therefore, more samples are not inherently bad, but they can
    become a drain.

- **uniquePlayers**
  - Means the number of different players who have played the game.
  - More/less unique players.
  - More unique players can bring more plays and playtime with them.
  - However, if uniquePlayers increases faster than Plays,
    retention decreases.
  - Therefore, attracting unique players is useful when those
    players also generate enough additional plays
### STATS

Now that we know the relationships between players and stats, we can
ask how to reinforce the statistics we want while preventing the
statistics we do not want from hindering us.

For every statistic, we consider not only how to change it, but what
other statistics naturally change alongside it.

A statistic should therefore not be considered a gain or drain in
isolation. What matters is the resulting change to the relationships
used by the formula.

- **GAIN**
  - **Plays**
    - **Method to Increase Stat**
      - Design Aspect
      - Player Action
      - Naturally Changed Stats
      - Tradeoff
      - Estimated Gains
  - **totalPlaytimeMs**

- **DRAIN**
  - **activeDays**
    - **Method to Minimize Its Effect**
      - Design Aspect
      - Player Action
      - Naturally Changed Stats
      - Tradeoff
      - Estimated Effect
  - **playtimeSamples**
  - **uniquePlayers**
