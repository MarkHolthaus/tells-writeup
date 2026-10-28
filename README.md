# Tells: how I tested an AI-built game, and the scoring flaw the test found

Tells is a poker roguelike that grades every decision you make against a solved strategy table, then lets you clear a level by playing accurately and beating the table. It is a game I am building, planned as a paid Steam release, so the code is private. This repo is the engineering story.

I design it, direct Claude Code to write the code, and test it. The test suite has 585 tests (480 game, 105 pipeline, Python `unittest`). The part worth reading is how I checked whether the game's scoring does what I designed it to do.

## The question

The clear condition has two halves: play accurately enough, and win the final heads-up fight. I wanted to know whether a player who plays *correctly* clears more than a player who just shoves all-in every hand. If they clear equally often, the grading is decorative.

## The method: paired-seed simulation

I built six simulated players with fixed styles: a solver-following grinder, a greedy analyst, an exploiter, a maniac (85% aggressive), a rusher and a passive "nit". Each played the same 200 deal seeds, 1,200 runs, no errors. Same seed means the first attempt's hole cards are identical for every player, so I could compare players deal against deal instead of against noise. I checked that the pairing held on 200 of 200 seeds instead of assuming it.

Two things I got wrong first and fixed:
- My first batch used different seeds per player, 13 each. The grinder cleared 0 of 13. That was luck. On the shared 200 seeds its clear rate is 15.5%, and 0 of 13 happens about 11% of the time at that rate.
- At 50 seeds no comparison was significant. At a clear rate near 15%, you need about 200 seeds to see one player clearing at twice another's rate.

## What it found

1. The correct player can clear. 15.5% at the first tier, with the best simulated player at 23.5%.
2. Passing the accuracy bar was not separating good play from bad. The maniac averages 0.50 accuracy per decision, yet cleared 21 of 200 first-tier attempts, and a sign test could not tell it apart from the correct player (10.5% vs 15.5%, p=0.13).
3. The cause was the scoring, not the bar. Each decision's weight depended on how much was at stake, and a player can choose how much is at stake. The maniac's play is mostly small blunders plus a few huge all-ins that happen to be close to correct. Weighting by stake gave it +0.168 accuracy while costing the correct player 0.023. All 21 maniac clears were below the bar on plain per-decision accuracy. No clear from a correct player was.
4. The stake term was counted twice. It made big-stake decisions grade more leniently and also made them weigh more, and the two effects compounded.

## Three fixes, priced before changing any code

I recomputed three candidate fixes from the recorded data, with the bar re-derived each time, so I could cost them with no code change:

| Fix | Maniac clears (tier 1) | Correct player's clears |
|---|---|---|
| Shipped scoring | 21 | 31 |
| Bar on weighted and unweighted accuracy | 0 | 31 |
| Cap each decision's weight at 1.5x the median | 2 | 31 |
| Unweighted accuracy only | 0 | 31 |

None of them changed what the intended player clears, because the bar is a fraction of that player's own measured baseline. I chose unweighted accuracy, and extended it to the payout base, since the leniency lived in the accuracy term. The measured result after the fix: maniac 21 to 0, rusher 9 to 0, correct players unchanged.

## What the test could not tell me

- Nothing about how the game feels on screen. That needs human playtests.
- The boss fight itself stayed random. A shove-every-hand policy wins it about 85% of the time against about 36% for the correct player. The accuracy bar, not the fight, is what separates them. I recorded that as a design question and did not change it.
- The numbers are for the first attempt at each tier, the only attempt every simulated player shares a seed for.

## What I took from it

When a metric can be influenced by the thing it measures, check it against a player who games it. Pair the comparison so noise cannot hide the effect. Price the fix before building it.
