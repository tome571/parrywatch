# parrywatch
Data Analysis to catch cheaters in the Valve game: Deadlock

A single-file, zero-dependency browser tool that screens your recent Deadlock lobbies for statistically improbable parry performance, investigates flagged accounts across their careers, and — for the cases that survive scrutiny — pulls tick-level evidence straight out of the match replay. Open parrywatch.html in a browser, paste a Steam ID, press OPEN CASE. No server, no install, no build step.

Data comes from the community-run deadlock-api.com (there is no official Valve API). An API key is optional and raises some limits, but the tool is designed to run keyless.

Flags are statistical leads, not accusations. The tool's entire architecture exists to disprove its own flags before it lets one stand. A person is behind every account.
-------
Original intent was to look at auto parrying players, using match metadata. 
After in-depth research into individual matches and player data, we're seeing 
that autoparry counterspell use is the most often found case in the wild. Working
updating the data analysis to account for tracking these cases as well. 

More to come!
