# Limits

What a live app's code can do in one request, what to build somewhere else, and what happens when a plan's allowance runs out.

## One Request

Each request to a site, and each scheduled run, gets at most 30 seconds of CPU time. Time spent waiting on a fetch or the database does not count, but work done in code does: parsing a large file, resizing images, or looping over thousands of rows. Each request can also make only a limited number of outbound fetches.

A request past either limit stops, and the visitor sees "This page took too long". A scheduled run past it stops too. Other requests to the site are not affected.

- Keep work per request small: page through data, and let SQL filter and sort rather than code.
- Split a long job across scheduled runs, each doing one batch and saving where it stopped.
- When a build passes but a page fails in use, read the server log with `read_site_logs` and `source: "server"`.

## Heavy Or Long Work

A site serves pages, APIs, scheduled jobs, and its own data. Waiting on AI, a fetch, or the database costs no CPU time, so a request or scheduled run can make several AI calls. Work that needs more CPU than one request or run gets, or that keeps going long after a visitor would stop waiting, runs another way:

- An AI agent that keeps working on its own for a long time.
- Video or audio processing, large file conversion, or anything else that needs minutes of CPU.

Build that part outside Artifen and have the site call it, keeping its keys with `set_site_secret`, or split it into batches across scheduled runs. Tell the person which part does not fit and why, and offer the closest way that does.

## Plan Allowances

Each workspace's plan sets allowances for what its sites keep and use. Never state a number from memory. The workspace's payer sees their usage, across the workspaces they pay for, in the Artifen app under Settings, then Usage, and changes the plan under Settings, then Billing.

- **A tool call past an allowance** answers `quota_exceeded` and changes nothing. `details.allowance` names the allowance, and `details` may add numbers such as `limit`, `used`, and `renews_at`. Tell the person what the message says. Remove things only with their agreement.
- **Sites past the plan's site allowance**, the newest first, refuse changes, publishing, and upload links that way, and their server features pause. They still serve their pages, and reading and deleting them still work. Restoring a site from Trash needs room under the allowance.
- **A site whose server use or database size is past its plan** keeps serving its pages, and its server features pause. Server requests are counted across every site the payer's plan covers, so once they run out, those sites pause together. A form post or any other request that is not a GET, HEAD, or OPTIONS, and opening a live query, answer "This can't be done right now", and AI and email calls in its code fail with a message that the allowance is used up. Site code should catch that and show a plain message instead of breaking the page. The pause ends when the allowance renews or the plan grows, or, when the site's database is too large, once a published migration deletes enough data that the live and preview databases together fit the allowance again.
- **Changes to a page** (a site with no `package.json`) count no build, and no plan limits them. Only a script that makes hundreds of changes in a day is stopped, with `quota_exceeded` and the day it renews. Publishing counts no build either.
- **Live app builds a day** are an allowance. Each build of a live app that runs counts one, a failed one included, and may take 5 minutes, whose time draws on the usage budget. Batch related edits into one change, and read the build's errors before sending the next.
- **Once the day's live app builds or the budget are used**, a change to a live app, or one that adds `package.json` to a page, is refused before it is saved. A push, a restore, or a change from GitHub is saved, and its build fails with the plan's words, so the address keeps what it served until a build runs after the allowance renews or the plan grows.
- **A few of one account's live app builds run at once.** The rest wait their turn, and one that waits too long is built again a minute later.
- **A new account** has limits, during its first days, on how many sites it creates in a day and how many it opens beyond the people invited to them. Past either it answers `rate_limited` with `retry_after_ms`, or `limit_reached` with the date the limit lifts.
