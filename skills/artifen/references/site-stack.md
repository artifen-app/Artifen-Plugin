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

Each site has its own SQLite database, and its preview has a separate copy seeded from live. Use it from server code: loaders, actions, API routes, and scheduled runs.

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

const { rows, status } = useLiveQuery<Guest>("guests", {}, { initialRows: loaderData.guests });
```

Writes through `database()` reach every open page at once. `status` is `live` while changes arrive, and the hook reconnects after a dropped connection.

## Visitor Sign-In

Artifen signs visitors in for the site, with Google, a passkey, a password, or a guest name.

- `visitor()` returns the signed-in visitor, `{ id, name, kind }` with `kind` either `identity` or `guest`, or null. `name` can be null. The `id` is the same on every visit to this site and different on every other site. Store it to know whose rows are whose.
- Nothing tells site code whether a visitor owns the site, and anyone with the link can be the first to claim it. When a page should be the owner's alone, ask the person how they will prove it, such as a passphrase set with `set_site_secret` for both `live` and `preview`, and remember only a visitor whose `kind` is `identity`. The owner can also read the data with `query_site_data`, with no page at all.
- Link to `/__artifen/visitor/sign-in?return=/path` to sign someone in, and to `/__artifen/visitor/sign-out` to sign them out.
- `site()` returns `{ siteId, slot, host }` for the address serving the request. It and `visitor()` are null in local development.

## Files, Email, AI, Payments, And Comments

```ts
import { ai, comments, email, files, payments } from "@artifen/site";
```

- `files.put(key, body, { contentType })`, `files.get(key)`, `files.delete(...keys)`, and `files.list({ prefix })` keep files the site stores, such as visitors' uploads or video. Each is served on the site's own address at `/__artifen/files/<key>`, with range requests for video.
- `email.send({ to, subject, html, text, replyTo })` sends from the site. From the preview, every message goes to the site's owner instead.
- `ai.generate({ system, messages: [{ role, content }], maxTokens })` returns `{ text, usage }` on the site's AI allowance.
- `payments.createCheckout({ lineItems: [{ name, amountCents, currency, quantity }], successUrl, cancelUrl, customerEmail, metadata })` returns `{ id, url }`, a checkout page to send the visitor to. `payments.getCheckout(id)` says whether it was paid. `payments.refund({ paymentIntentId, amountCents })` refunds. A workspace Owner connects the workspace's payments account first, in the Artifen app under Settings, then Payments. The preview takes test payments.
- `comments.list({ page })` and `comments.create({ page, body })` read and add comments on the site's pages. Adding one needs a signed-in visitor.

When the site's plan has used up an allowance, these answer with an error, and the site's pages keep serving.

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

Add cron expressions to `triggers.crons` in `wrangler.jsonc`, such as `"crons": ["0 8 * * *"]`, and put the work in the `scheduled` handler in `workers/app.ts`. Times are UTC.

## APIs And An MCP Endpoint

An API is a resource route: a module with a `loader` for GET or an `action` for other methods, no component, listed in `app/routes.ts`.

The starting app serves its own MCP server at `/mcp`. Add tools in `app/mcp.server.ts` with `server.registerTool(name, { description, inputSchema }, handler)`, using a zod object for `inputSchema`, and keep the `createMcpHandler` call and its allowed hosts as they are. Other agents reach it at `<site url>/mcp`. A private site asks every request to sign in, so the endpoint works for agents once the site is open to anyone with the link, or public.

## Page Titles And Link Previews

Each route's `meta` export sets its tab title, description, and link preview tags, such as `[{ title: SITE_TITLE }, { name: "description", content: "..." }, { property: "og:image", content: "/og.png" }]`.
