# The Site Stack

A site created with `start_from: "template"` is a React Router v8 app in framework mode, rendered on the server, styled with Tailwind, with `@artifen/site` for everything Artifen provides. The site's own `AGENTS.md` covers the same ground in brief.

## What Each Build Checks

Every change installs packages, generates route types, type-checks, lints, checks the routes, the imports, and the migrations, and builds. Each problem comes back in `build.errors` with its `step` (`install`, `typegen`, `typecheck`, `lint`, `build`, `routes`, `imports`, or `migrations`) and the file and line. The address keeps serving the last version that built.

- Import from `react-router`, and from `@react-router/dev` in config files. `react-router-dom` and `@remix-run/*` are older packages, and a build that imports them fails at `imports`.
- Every route module in `app/routes` must be listed in `app/routes.ts`, or the build fails at `routes`.
- Route modules import their types from `./+types/<module name>`, which the build generates from `app/routes.ts`, such as `import type { Route } from "./+types/home"`.
- Add an npm package by adding it to `dependencies` in `package.json`. The build installs it. Keep `pnpm-lock.yaml` in the site, and keep the tools the starting `package.json` lists, since the build refuses a project without them.
- `wrangler.jsonc` configures the site's server. Leave its database binding, `migrations` entry, and `ARTIFEN_SITE_HOSTS` as they are, and add no bindings to it, such as storage, queues, or AI: a build that declares them fails. Everything those would give comes from `@artifen/site`.
- Paths under `/__artifen/` belong to Artifen.

## Project Layout

| Path | What it holds |
| --- | --- |
| `app/routes.ts` | Every route: `index("routes/home.tsx")`, `route("about", "routes/about.tsx")` |
| `app/routes/*.tsx` | Pages, each with an optional `loader`, `action`, and `meta`. A module with a `loader` or `action` and no component is an API route |
| `app/root.tsx` | The document around every page |
| `app/site.ts` | `SITE_TITLE`, written from the site's title |
| `app/live.server.ts` | The site's live queries |
| `app/mcp.server.ts`, `app/routes/mcp.ts` | The site's own MCP server at `/mcp` |
| `migrations/*.sql` | Database migrations |
| `workers/app.ts` | The server entry, with the `scheduled` handler |

The starting notes page is a sample to replace.

## Database

Each site has its own SQLite database, and its preview has a separate copy seeded from live. Use it from server code: loaders, actions, API routes, and scheduled runs. Whatever visitors save for others to see goes here, never into the site's own files.

```ts
import { database } from "@artifen/site";

const rows = await database().query<{ id: number; name: string }>("SELECT id, name FROM guests ORDER BY id");
const one = await database().first<{ id: number }>("SELECT id FROM guests WHERE name = ?", name);
const { changes, rows: inserted } = await database().run("INSERT INTO guests (name) VALUES (?) RETURNING id", name);
await database().batch([{ sql: "DELETE FROM rsvps WHERE guest_id = ?", params: [id] }, { sql: "DELETE FROM guests WHERE id = ?", params: [id] }]);
```

`batch` runs its statements in one transaction.

Change tables only with migrations: a new SQL file in `migrations/` for each change, such as `migrations/0002_rsvps.sql`. They apply in name order, each once, when a version goes live, and a build fails when one does not apply after the ones before it. Never edit a migration that already ran, and leave transaction statements, `ATTACH`, `VACUUM`, and `PRAGMA` out of them.

## Live Updates

Define each live query on the server by name, and pages open it by that name with arguments, so a page never sends SQL. Give live rows an `id` column, so a page receives only the rows that changed.

```ts
// app/live.server.ts
import { liveQuery } from "@artifen/site";

export const liveQueries = {
  guests: liveQuery(() => ({ sql: "SELECT id, name FROM guests ORDER BY id" })),
  myRsvps: liveQuery((_args: Record<string, never>, { visitor }) => ({
    sql: "SELECT id, answer FROM rsvps WHERE visitor_id = ?",
    params: [visitor?.id ?? ""]
  }))
};
```

The starting app already serves them from `app/routes/live.ts`, listed in `app/routes.ts` as `route("api/live", "routes/live.ts")`. In a component:

```tsx
import { useLiveQuery } from "@artifen/site/react";

const { rows, status, error } = useLiveQuery<Guest>("guests", {}, { initialRows: loaderData.guests });
```

Writes through `database()` reach every open page at once. `status` is `live` while changes arrive, the hook reconnects after a dropped connection, and `error` holds a message to show when the query failed.

## Who Is Visiting

`visitor()` returns the person on the page, `{ id, name, kind, role }`, or null for someone not identified:

| `kind` | Who | `role` |
| --- | --- | --- |
| `member` | Someone with a role on the site, such as its owner, its workspace's members, or people it was shared with, signed in with their Artifen account | `owner`, `editor`, `commenter`, or `viewer` |
| `identity` | Someone signed in through the site's visitor sign-in, with Google, a passkey, or a password | `visitor` |
| `guest` | A name given without an account | `visitor` |

- The `id` is the same on every visit to this site and different on every other site. Store it to know whose rows are whose. A member who also uses visitor sign-in has a different `id` for each, and while both are signed in the site sees the member. `name` can be null.
- A page or action meant for the owner checks the role it needs in its loader or action, such as `visitor()?.role === "owner"`. A live query answers as the person it opened with until it reconnects, which suits rows filtered to each visitor, so data only some roles may see comes from a loader or an action instead.
- On a site open to anyone, the owner arrives as an unidentified visitor until they sign in as a member. Show anyone whose `role` is `visitor`, or who is null, a link or button to `/__artifen/member?return=/path`, which signs members in with Artifen when they click it. On a private site and on a preview, everyone who gets in is a member and needs no member link, though `visitor()` can still be null there: treat null as someone not identified.
- Link to `/__artifen/visitor/sign-in?return=/path` to sign a visitor in, and to `/__artifen/visitor/sign-out` to sign them out.
- `site()` returns `{ siteId, slot, host, appUrl }` for the address serving the request. `appUrl` is the site's page in the Artifen app, for a link that takes its owner there.

## Files, Email, AI, Payments, And Comments

```ts
import { ai, comments, email, files, payments } from "@artifen/site";
```

- `files.put(key, body, { contentType })`, `files.get(key)`, `files.delete(...keys)`, and `files.list({ prefix })` keep files the site stores, such as visitors' uploads or video. Each is served on the site's own address at `/__artifen/files/<key>`, with range requests for video.
- `email.send({ to, subject, html, text, replyTo })` sends from the site. From the preview, every message goes to the site's owner instead.
- `ai.generate({ system, messages: [{ role, content }], maxTokens })` returns `{ text, usage }` on the site's AI allowance.
- `payments.createCheckout({ lineItems: [{ name, amountCents, currency, quantity }], successUrl, cancelUrl, customerEmail, metadata })` returns `{ id, url }`, a checkout page to send the visitor to. `payments.getCheckout(id)` says whether it was paid. A workspace Owner connects the workspace's payments account first, in the Artifen app under Settings, then Payments. Real payments are not open on Artifen yet, so at the site's own address, `site.url`, both fail with a message saying so, whether the site is private or shared. A preview takes test payments, and a site has one only once it is shared beyond the people invited to it. Tell the person this when they ask for payments, and leave the choice to share to them.
- `comments.list({ page })` and `comments.create({ page, body })` read and add comments on the site's pages. Adding one needs someone signed in: a visitor comments under the name they signed in with, and a member under their own name, except a member whose role is `viewer`, who can comment only through visitor sign-in on a site open to everyone. Otherwise `comments.create` throws a message to show the person.

When the site's plan has used up an allowance, these answer with an error, and the site's pages keep serving.

Each request and scheduled run gets at most 30 seconds of CPU time. Waiting on AI, a fetch, or the database does not count, but heavy processing does.

## Secrets

Secrets set with `set_site_secret` arrive in `env` by name:

```ts
import { env } from "cloudflare:workers";

const key = env.MAPS_API_KEY;
```

Type checks know only what `wrangler.jsonc` declares, so declare each secret's name in a file such as `app/secrets.d.ts`:

```ts
declare namespace Cloudflare {
  interface Env {
    MAPS_API_KEY: string;
  }
}
```

## Scheduled Jobs

Add cron expressions to `triggers.crons` in `wrangler.jsonc`, such as `"crons": ["0 8 * * *"]`, and put the work in the `scheduled` handler in `workers/app.ts`. Times are UTC. Write each expression as five fields of numbers, `*`, `,`, `-`, and `/`, since names such as `MON-FRI` and shorthands such as `@daily` never run. A preview runs its schedules too, against its own database.

## APIs And An MCP Endpoint

An API is a resource route: a module with a `loader` for GET or an `action` for other methods, no component, listed in `app/routes.ts`.

The starting app serves its own MCP server at `/mcp`. Add tools in `app/mcp.server.ts` with `server.registerTool(name, { description, inputSchema }, handler)`, using a zod object for `inputSchema`, and keep the `createMcpHandler` call and its allowed hosts as they are. Other agents reach it at `<site url>/mcp`. A private site asks every request to sign in, so the endpoint works for agents once the site is open to anyone with the link, or public. Then anyone who can open the site can call every tool, so a tool that changes or reveals data checks who is calling first, such as with a key set by `set_site_secret`.

## Page Titles And Link Previews

Each route's `meta` export sets its tab title, description, and link preview tags, such as `[{ title: SITE_TITLE }, { name: "description", content: "..." }, { property: "og:image", content: "/og.png" }]`. `SITE_TITLE` in `app/site.ts` starts as the site's title. Put the icon and the preview image in a `public/` folder at the site's root, which the starting app does not have yet, where `public/icon.svg` is served at `/icon.svg`, and add a `links` export to `app/root.tsx`, such as `export const links = () => [{ rel: "icon", href: "/icon.svg" }];`, with your own file names. Link previews need an absolute `og:image` address, so build it in the loader from `site().host`.
