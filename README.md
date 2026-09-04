# Tosh Protocol — status page

One static file, `index.html`. No build, no dependencies, no CI. GitHub Pages
serves it straight from `main`.

## Why this is a separate repository

Because a status page hosted on the thing whose status it reports is worthless
in the incident that matters most. `docs/INCIDENT_RESPONSE.md` §0 lists
"malicious code in the frontend bundle" as a P0 example; a page served from the
same Vercel project, the same domain and the same deploy pipeline would be down
or compromised alongside it. This repository shares nothing with the
application except the chain it reads.

## Updating it during an incident

You can do this from a phone browser with no laptop and no command line, which
is the only requirement that actually mattered when choosing where to host it.

1. Open `index.html` on github.com and tap the pencil.
2. Change the three lines under **EDIT THESE THREE LINES**:

   ```js
   const STATUS  = 'paused'                      // operational | investigating | paused | resolved
   const DETAIL  = 'Paused at block 112,684,518.' // optional, one line
   const UPDATED = '2026-09-04 03:12 UTC'
   ```

3. Commit to `main`. Pages rebuilds in about a minute.

Do not write incident prose here under stress. Each `STATUS` already carries
the wording agreed in advance, and the `paused` text is the P0 playbook's Step
4 message verbatim. `DETAIL` is there for the one specific fact — a block
number, a scope limit — that the pre-written copy cannot know. If you reword
the playbook, reword `COPY` in the same commit, because a status page that
contradicts the runbook is how a responder starts improvising.

## What the page reads by itself

The "read from the chain just now" block is not typed by a human. The browser
calls the chain's public RPC and prints `paused()` and the block height, so
that section cannot go stale while the banner is being forgotten. If the two
disagree the page says so explicitly, in both directions, and names the chain
as the authority — a page updated ahead of the transaction landing and a page
nobody updated are both real failure modes, and quietly showing one number is
worse than showing the contradiction.

An unreachable RPC prints "could not reach the RPC" rather than `false`. An
endpoint that cannot be reached is not evidence of health.

## Chain cutover

`CHAIN` at the top of the script points at the rehearsal chain (46630), which
is what the frontend also reads until PM-C1. The mainnet block sits commented
directly beneath it. Swapping them is PM-C7's sibling and belongs in the same
sitting.
