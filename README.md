# PARRYWATCH — Bureau of Melee Forensics

A single-file, zero-dependency browser tool that screens your recent Deadlock lobbies for statistically improbable parry performance, investigates flagged accounts across their careers, and for the cases that survive scrutiny, it pulls tick-level evidence straight out of the match replay. Open `parrywatch.html` in a browser, paste a Steam ID, press **OPEN CASE**. No server, no install, no build step.

Data comes from the community-run [deadlock-api.com](https://deadlock-api.com) (there is no official Valve API). An API key is optional and raises some limits, but the tool is designed to run keyless.

**Flags are statistical leads, not accusations.** The tool's entire architecture exists to *disprove* its own flags before it lets one stand. A person is behind every account.

## Why parries?

The parry is one of Deadlock's purest reaction-time mechanics: a melee attack is coming, you have a fraction of a second to press the button, and the match metadata keeps score. `Parry Success` and `Parry Miss` are recorded per player in every match's custom stats. The fact they are there makes it uniquely auditable. The data analyst side of me said, Hmm, what kind of insights can we get from all of that data? 

Calibration from live investigation (July 2026): humans land roughly 30% of attempted parries, selectivity ranges ~10–50%, and misses often outnumber successes. Whiffing in anticipation and missing is *normal*. Sustained high selectivity at volume, or repeated zero-miss performances, is not normal! As a bonus, a parry decision leaves a clean paper trail so we can audit. 

An auto-parry script has a few signatures no human produces: it reacts faster than human perception allows, and it parries attacks that give no usable warning. One common behavior, but not always, is many of them it parries at the last possible instant. The tool is built to measure all three.

## The funnel

The tool runs a four-phase funnel. Each phase is cheap relative to the ones that follow. Each exists to keep the expensive phases used the least, and only on the most suspicious cases.

### Phase I–II: Lobby screen and flags

Pulls your recent match history (default 50 matches), fetches each match's metadata, and reads the parry counters for *all twelve players in every lobby* That's right, you're screening everyone you've played with or against, not just the one guy you suspect. Street Brawl and bot matches are excluded by default (parry stats are a little more muddy in those modes from my current data observations; toggleable).

An account gets flagged when its numbers clear thresholds calibrated against the human baseline: 
- 70%+ selectivity over 12+ attempts 
- 60%+ over 25+ attempts 
- zero misses at 6+ successes (roughly a 1-in-4,000 event against the 30% baseline). 

Flags are ranked by severity (`selectivity × log₂(attempts+1)`) so investigation budget goes to the worst offenders first.

### Phase III: Career investigation

A single hot lobby means nothing, and is actually fairly common. The funnel was created because of a player on a 6-of-8 match that turned out to be their career-best game. Therefore, the approach was look at those flagged above and ask, does the success rate for their parries continue over multiple games? So every flagged performance gets a career pull: the account's stored match history, analyzed in stages (a quick pass over the newest 12 matches exonerates most people immediately; anything unclear extends to 40, then 100 matches). The question is *persistence*: does the anomaly repeat across weeks of play, or was it one good night?

This phase is also where **innocent explanations get subtracted**, because two legitimate things inflate the parry counter:

**Counterspell** routes ability-blocks through the parry button. Those "parries" answer loud, telegraphed on-screen warnings (anyone ever miss a knockdown counterspell?). Because of that high hit rate, they say nothing about *melee-parry* skill, so matches where the account bought Counterspell are excluded from the melee verdict. **But set aside is not exonerated.** After analyzing 10,000+ accounts.. Counterspell auto-block is the **dominant cheat pattern in upper MMR brackets** A little online search shows that hacks are often ones that activate only after CS is purchased and trigger only on abilities that specifically target the player. To account for this, Counterspell matches route to their own audit track (below) instead of quietly vanishing from the analysis. 

**Apollo's Riposte** writes successes to the counter without the parry button being pressed at all. This was discovered via population analysis: every player of that hero shows inflated numbers, which is the fingerprint of kit contamination rather than individual cheating. Riposte-hero matches are likewise set aside. 

**Indomitable** was tested the same way and cleared: it plays the parry sound but does not write to the counters (verified empirically: Indomitable-only matches pooled 32%, virtually indistinguishable from the 30% clean baseline, across 28 careers). It's annotated for reference, but counts toward the clean career.

The verdicts: CLEAN / WATCH / CONFIRM are computed only from what's left after set-asides, where high numbers have no innocent explanation. Two escalation guards keep the set-asides honest. First, if the clean sample is too small to judge *and* the raw career is anomalous, the case escalates to the final stage of timing audits instead of being dropped. Second, the **Counterspell profile**: each career is profiled for CS buy rate, hero spread, buy timing, and counter selectivity within Counterspell matches. Buying Counterspell every game, early, across many heroes (including ones it makes no sense on for typical play) at high counter selectivity is the career signature of the auto-block cheat, and it escalates the account to CONFIRM with a Counterspell-specific note. Profile thresholds are a work in progress, pending population calibration.

A depth caveat the tool is honest about: career persistence can distinguish a streak from a trait, but it cannot distinguish an honest high-end player from a *humanized* script at the same rate. Only the timing audit answers mechanism.

### Phase IV: Timing audit (the autopsy)

For confirmed cases, the tool pulls tick-level data from the actual match replay using the API's DemoFusion engine (SQL queries executed against the `.dem` file server-side). One combined query per match retrieves five tables: the melee-parry ability state, melee attack instances, the player roster, the game-clock anchors, and SpellShield (Counterspell) activations. This can also be manually done if you have suspicions of an account and want to see the raw data breakdown. Maybe they run a hot 54% all time? Either they are a true pro or a part-time auto parry user and the match stats aren't enough to tell. Run the audit and look at the timings!

From those, per player who ever pressed parry:

**Reaction time** — how long after the attacker started their windup the parry was pressed. Visual reaction is ~200ms; players who swap in a louder heavy-melee sound cue legitimately react by ear at ~150–170ms. A *typical* value under 150ms across 3+ parries is flagged — faster than any honest human, even by ear. The threshold deliberately sits below the fastest honest player, not the average one.

**Too-fast ("phantom") presses** — successes with no visible press row within 1.2s beforehand. A human's anticipation parry serializes as its own row before the success resets it; a last-instant reactive trigger collapses into the same update interval and leaves nothing. Low phantom share is reasonably human behavior. However, ≥50% at 4+ successes is the auto-parry tell.

**Counterspell audit** — Counterspell has no separate activation: one parry press has three outcomes: melee parried, ability blocked (a parry success that starts the SpellShield cooldown), or a plain whiff. The replay therefore records successful blocks only, the miss pool is *shared* between melee and spell intent, and block accuracy convicts no one because honest blocks are often from loud telegraphed casts and rarely miss (the logic behind the melee set-aside). This reframes the funnel's stance: a one-off CS match with big counter numbers is *explained*; a career trend of buying CS nearly every game is *suspicious* and gets audited rather than handwaved. The audit's job is to decide whether replay review is warranted, using two mechanical signals that don't depend on knowing what was blocked. **No-press blocks**: a human's button press serializes as its own row before the block, while an injected input collapses into the block's update. This reads as the spell-domain twin of the too-fast column (flags at 50%+ of 3+ blocks). **Press-to-block gap**: a human holds the parry window early and eats the gap between press and impact, while impact-timed injection lands the block almost on the press. This is a tiny median gap with tiny variance (flags under 50ms median at 3+ blocks). Both thresholds are speculation now, and pending calibration. When either fires, the suspect note stamps REPLAY REVIEW RECOMMENDED with every block timestamped at the scoreboard clock, and the review question per block is *could they perceive the cast?*. Important to understand when reviewing that blocking telegraphed casts and predictable DOT ticks is skill; repeatedly blocking abilities that specifically target the player, arriving from fog with no audio cue, is impossible information. For Counterspell-profile accounts, the audit walker prioritizes Counterspell matches when picking which replay to pull, since the auto-block cheat appears to be dormant in matches without the item. The machine-evidence upgrade (planned) is pulling enemy ability-cast timing from the demo so cast-to-block reaction time can be computed directly.

**Attack-type breakdown** — every success is matched to the nearest opposing melee instance and classified by attack type. Light jabs give ~0.1s of warning: parrying one is luck or an already-open window; parrying several *on reaction* is not humanly possible (the audit flags at 2+, with scoreboard-clock timestamps so you can verify each one in the replay viewer. Note the attack matching is approximate, so the replay is the final word). Successes coinciding with a SpellShield cooldown start (±0.5s) are Counterspell spell-blocks and are routed to the "unclear" column, never the no-warning column.

The audit also reconciles metadata against replay: if the match counter credits an account more successes than the replay's parry ability shows landed by button, the gap is a Counterspell/kit usage meter — and if a flagged account doesn't appear in the replay's parry data *at all*, every counted parry came from kit sources. Orphan presses (button pressed, no melee parried) are listed with timestamps and a manual-review guide, since blocking projectiles arriving from fog with no audio cue — repeatedly — is the spell-domain equivalent of impossible information.

Each audit also feeds a per-hero windup calibration table (`hero:attack-type → windup ms`), accumulated across every autopsy and exported with the screen JSON, because melee timing varies per hero (fast-light heroes like Billy have genuinely different parry-exposure profiles) and the assets API exposes melee damage but not timing — demos are the only timing source.

## Clock calibration

Replay events are recorded in demo ticks; humans scrub replays by the scoreboard clock. The conversion has two subtle failure modes, both handled: the game-rules clock property stores its update moment as a *server* tick, a different counter from the demo tick the queries return (comparing them directly shifts every timestamp by a constant offset), and the tick rate itself varies slightly per replay (extrapolating at a hardcoded 60 drifts with elapsed time). The tool therefore re-anchors each clock value at the first demo tick it was observed, measures the replay's true ticks/second by regressing demo ticks against melee game-time stamps (hundreds of samples per match), and interpolates piecewise between anchors — which also freezes the clock correctly across pauses. Each audit logs its calibration line (measured rate, anchor count, demo-vs-server tick offset) so conversions are verifiable.

## Budget, rate limits, and being kind to the API

The tool is a guest on community infrastructure and behaves like one. Requests are paced by an adaptive global gap (~9 launches/s baseline, halving speed on any 429 and recovering slowly), metadata fetches run through a small concurrent worker pool, and every fetched match is distilled to a ~2KB record and cached — re-runs skip the fetch phase entirely, and the cache is exportable/importable across sessions.

Demo queries are the scarce resource (20 submissions/hour without a key). Each audit costs exactly one submission — the five tables ride a single UNION ALL query with a source-discriminator column, padded to a common column shape and demuxed client-side — so the default budget of 18 covers 18 audited matches. The audit walker tries up to four candidate matches per account (newest spikes first), because replays only exist on Valve's side once someone has requested them: a missing replay may simply never have been minted, and watching the match once in the in-game replay viewer often fixes that.

Match history is served from the API's stored (ClickHouse) data and is effectively unlimited; only the Steam-fresh refresh path for bot-friend accounts is capped hourly, and when that cap is hit the API returns the stored history in the 429 body — which the tool uses rather than failing. One coverage note: stored history only knows matches the community ingested, so a career can be thinner than reality; the case board flags fresh/thin accounts as context.

## Outputs

The case board renders each flagged account with its raw and clean selectivity, unexplained hot matches, a career bar-strip (height = attempts, color = classification, hover for per-match detail), and its verdict stamp. Exports: the screen data as JSON (including the windup calibration table), a Markdown report of flags and verdicts, and the session cache for cheap re-runs. The field log records every decision the funnel made, in order, so a run can be audited after the fact.

## Interpreting results

A CONFIRM stamp is the funnel's opinion; the timing audit is the evidence. The intended reading order is: does the clean career (after set-asides) show persistent anomaly against baseline human parry rates? If so,  does the replay show a mechanical signature in the data (sub-human reactions, high phantom share, no-warning parries)? If so, do the flagged moments survive your own eyes in the replay viewer at the listed timestamps? Anything that fails a step upstream never reaches the step below it, and the tool says so using data, rather than guessing.
