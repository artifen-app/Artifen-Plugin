---
name: "artifen"
description: "Use when building, publishing, updating, or fixing a site, web app, page, game, dashboard, or slideshow on Artifen (artifen.app, artifen.site), and whenever a task calls Artifen's tools such as create_site, write_site_file, or publish_site, or the `npx artifen` CLI. Also use whenever the person asks for an artifact, page, app, or slideshow they can share, send, or open on another device while Artifen is connected, even without naming Artifen. Teaches when to build on Artifen, how to start, write, and change a site from plain files to a full-stack app, publish it, and recover from errors."
license: "Apache-2.0"
metadata:
  author: "Leeor Nahum"
  version: "0.1.1"
---

# Artifen

Artifen puts what you build online as a site with its own address on `artifen.site`, from one HTML page to a full-stack app with a database, sign-in, files, email, AI, and payments. The tools, the `npx artifen` command line, and the HTTP API reach the same sites under the same rules.

## Build Here When The Person Wants To Share

When Artifen is connected and the person asks for a page, slideshow, game, dashboard, or app to host, share, send, or keep, or to open on a phone, build it as an Artifen site with `create_site`, since a site has an address the person can give to others and keeps working after the chat ends, and give them that address. When the person names something else, such as a Claude artifact, a canvas, or a file, do that instead, and never replace what they asked for.

When the tools are missing and you cannot run `npx artifen`, tell the person they can connect Artifen to this app by following http://localhost:3000/connect.

## Create A Site

1. Call `list_workspaces` and pick one where `can_create_sites` is true. When several qualify and the person has not said which, ask, since the workspace owns the site and decides who else sees it.
2. Choose where it starts, `start_from`. People see two kinds of site, pages and live apps:

   | The site | `start_from` |
   | --- | --- |
   | A page: a slideshow, a game, a calculator, a list people check off, a poll, a sign-up sheet. It runs in the browser, and it can save entries through Artifen's data store and sign people in | `files`: plain HTML, CSS, and JavaScript, served exactly as sent, with no build |
   | A live app: it has logic of its own, such as pages made on request, its own database tables and queries beyond saving entries, payments, scheduled jobs, email, AI, or its own API or MCP endpoint | `template`: Artifen's starting app, React Router with server rendering, Tailwind, a database, and live updates |

   Data a page saves for other people, such as check-offs, votes, notes, or sign-ups, goes in the data store, as [page data](references/page-data.md) shows, and the site stays a page. Never write it into the site's own files, where your next edit would collide with it. What one visitor keeps only for themselves can stay in their browser's own storage.

   With the person, say page or live app, never template, server code, or database. Pages and live apps both count toward a plan's sites, and only live apps toward its live apps. Before building a live app, or a change that turns a page into one, tell the person in plain words that it becomes one of their plan's live apps, that the Artifen app shows how many their plan has in use, and that once the plan's number is reached, the live app used least recently is archived to make room. Prefer a page when it does what they asked.

3. Call `create_site`.
   - Give every site, in its own code, a page title and a meta description that name it in real words, and an icon, with `og:title` and `og:description` matching them. Add a share preview image (`og:image`) only when the page is made to be shared or the person asks, since Artifen otherwise shows a picture of the site's home page. The `title` given to `create_site` is only the starting name: after each build that is published, the home page's title and meta description become the site's title and description in Artifen, and its address follows the title. So rename a site by changing its page title, since no tool sets them. A title or description the owner edits in the app is kept through later builds. With `files`, put the `<title>`, `<meta>`, and `<link rel="icon">` tags in each page beside the image files. With `template`, the site stack shows where.
   - With `files`, send `index.html` and every file the site needs in one call. Keep `package.json` out of the root, since it turns the site into an app that must build, and keep out TypeScript, JSX, Sass, and other files only a build can serve, which fail the build. There is no fallback page, so a client-side router needs hash routes or one HTML file per route. A `404.html` becomes the not-found page.
   - A page becomes a live app at its own address with one `apply_site_patch` call with `add_starting_project: true`: Artifen's starting app is added, titled with the site's title, and the same patch moves the page's files under `public/` with `*** Move to:` hunks. Read [the site stack](references/site-stack.md) before doing it, and tell the person first, as above.
   - With `template`, files sent with the create replace the template's files at the same paths, so create it with none or only new files, then read the new site's `AGENTS.md`, `app/routes.ts`, and the sample pages before changing them. Read [the site stack](references/site-stack.md) before writing any code for it.
4. Check `change.build` in the result (see Every Change Builds), then give the person `site.url`.

Each `create_site` call makes another site, so after a failure that may have landed, call `list_sites` before trying again, and carry on with the site if it exists.

## Who Can Open It

A new site opens only for the members of its workspace and anyone invited to it. Sharing, roles, workspace members, custom domains, GitHub sync, billing, and deleting a site for good are changed in the Artifen app, never by a tool. A tool call that reaches one of them answers `app_only`.

- Tell the person who can open the site now, from `site.access_level` (`private` is the workspace's members and invited people, `link` is anyone with the link, `public` is anyone, and found by search engines).
- Tell them where to change it: the site's page in the Artifen app, `site.app_url`, under Share. They can invite people by email, open it to anyone with the link, make it public, or let a room of people join with a QR code.
- When you operate a browser the person is signed in to and they ask you to share the site, you may do it through that same Share dialog.

## Change A Site

To find a site the person means, call `list_sites` with words from it as `query`. Then start with `read_site` and no `path`: it returns the site's addresses, your capabilities, the published and preview versions, the latest build, secret names, and the file list.

- **Exact edits first.** `edit_site_file` replaces `old_string`, which must appear exactly once in the file's current content. Copy it from a fresh read, never from memory.
- **Several files together.** `apply_site_patch` adds and changes files as one version, all or nothing. It never deletes. Use `delete_site_files` with exact paths for that.
- **Whole files need a full read.** `write_site_file` replaces an existing file only after you read all of it, following `next_offset` until it is null, and refuses one that changed since (`conflict`). A read made in another conversation through the same app can count as yours, so read the file again just before replacing it.
- **One change per intent.** Put every file of one change in one call. Each change makes a version and a build, and pages show it on their next load once the build succeeds.

## Every Change Builds

Every change returns `build` with a `status`:

- `succeeded`: `url` in the result serves the change.
- `failed`: the change is saved as a version, and the address keeps serving its last working version. Each entry in `build.errors` names the `step` that failed, the `file`, `line`, and `message`. Fix them and send the fix as the next change. When an error's `rule` is `safe_browsing_checks_used`, the day's safe-browsing checks are used and no change fixes it: tell the person, and send the change again after midnight UTC.
- `queued` or `running`: not finished. Check `latest_build` from `read_site`, or `read_site_logs` with `source: "build"`, before telling the person it works.
- `superseded`: this change was not deployed by this build. When `build.covered_by` is null, a newer change reached the address first. Otherwise `covered_by` names the build that carries this change, with that build's status:
  - `succeeded`: the address serves the change.
  - `failed`: read that build's errors with `read_site_logs` and `source: "build"`, and send a fix as the next change.
  - `queued` or `running`: read the site again with `read_site` until its build finishes, before telling the person it works.
  - `superseded`: a newer build took its place. Read `latest_build` from `read_site`.

## Files That Are Not Text

| The file | Send it as |
| --- | --- |
| Text, such as HTML, CSS, JavaScript, JSON, SVG, or Markdown | `content`, its full text |
| At a public `https://` URL that answers with the file itself | `source_url`, which Artifen fetches |
| Attached to the conversation, in a host that hands attachments to tools | `attachment_index`, its position in `attachments` |
| On the person's computer, when you cannot run commands | A link from `create_site_upload`, which the person opens to choose the file. It lands as a new version |
| On a computer where you can run commands, or a whole folder | The command line, as [the shell reference](references/shell.md) shows |

Images, fonts, audio, video, and PDFs never travel as `content`. Never put base64 or a `data:` URL in `content`, and never pass a local path or a host's file ID as `source_url`.

## Preview And Publish

Artifen adds a small Report site link at the bottom right of shared published pages, and a Preview badge at the bottom left of previews, so keep those corners free.

While a site is private, it has no preview, and every change is published at `site.url` at once. Once the person shares it beyond invited people, changes land in its preview instead, at `site.preview_url`, which shows a Preview badge and has its own copy of the database, its own secrets, email that reaches only the owner, and test payments. A preview starts with no secrets, so when a site that uses secrets is shared, set each one for `preview` too, with test keys. `publish_site` makes the preview the version served at the site's address and applies its database migrations to the published data, and if anything fails, the address keeps serving what it served before.

After changing a shared site, give the person `preview_url` to check, and publish when they want the change at the site's address. `nothing_to_publish` means the site is private, so its changes are already published, or the preview matches what is published.

## History

Every change is a version. `list_site_versions` lists them newest first, with bookmarks and which version each address serves. `read_site` with `version_id` and `path` reads a file as it was. `restore_site_version` puts the files back as they were at a version, as a new version, so nothing in between is lost. The database is not rolled back.

## Logs, Data, Secrets, And Comments

- `read_site_logs` reads a build's log (`source: "build"`) or what a live app's code logged (`source: "server"`), for the live site or its preview (`slot`).
- `query_site_data` runs one read-only SQL query on the published site's database or the preview's copy (`slot`: `live` or `preview`). For a page, that is what it saved through the data store. Data changes go through the site's own code and migrations.
- `set_site_secret` stores a key a live app's code reads by name, separately for the published site (`slot: "live"`) and its preview (`slot: "preview"`), which should hold test keys. A private site has only the published one, and `read_site`'s `secrets` shows which slot holds each name. When the person pastes a key into the chat, store it as they ask. Never repeat its value in a reply or write it into a file. `delete_site_secret` removes one.
- `list_site_comments` reads what people left on the site's pages, and `reply_to_site_comment` answers one as the person.

## Scripts And Other Agents On A Private Site

A site open only to its people, such as a private live app, answers a script or another agent that sends a site key in the `Artifen-Site-Key` header on the site's own address, as the person the key belongs to. The site's code sees that person as a member and never sees the key. Keep the site private for this rather than opening it to anyone with the link. Keys are for requests from scripts and servers: a browser page on another address cannot send one to a private site, and a key opens no live connection and posts no comment in the person's name.

- For a script that runs elsewhere or on a schedule, such as a job that pushes data every night, `create_site_key` makes a key with a name saying what it is for. It is shown only in that result. Give it to the person once, to store where the script reads it, such as the script's own secret store, and never write it into the site's files. It lasts 90 days unless `expires_in_days` says otherwise. `list_site_keys` shows each key with when it was last used, and `delete_site_key` ends one. A key you make this way ends if the person disconnects this agent app, so for a job that must outlive this connection, the person makes the key in the site's settings in the Artifen app.
- For requests you make yourself now, `create_temporary_site_key` gives a key for the next hour with nothing to keep. Make one and reuse it for the hour. On the command line, `npx artifen keys temporary <site>` prints the same key again until ten minutes before it expires.
- A key carries at most its person's role, which `role` can lower, such as `viewer` for a script that only reads. Each request is checked against that person's access at that moment, so removing them from the site ends their keys for good, and so does disconnecting the agent app that made them. Requests with keys are rate limited per key and per site. A refused key answers `site_key_invalid`.

## Delete And Restore

Call `delete_site` only when the person asks to delete the site itself. It moves the site to Trash and its address stops at once. Tell the person the date in `undo` until which it can come back, and that you or Trash in the Artifen app, under Settings, can bring it back. `restore_site` brings it back with its files, versions, data, and address, and so does Trash in the Artifen app. A site whose workspace was deleted comes back only through Trash in the app, where the person chooses its new workspace.

## Archive And Pin

Archiving tidies and deleting frees room. `update_site` with `archived: true` archives a site and loses nothing: it keeps its address and its pages still open, a page that saves data shows what it has but takes no new entries, and a live app's own logic stops running, so its saved pages open but their buttons and forms do nothing, and it stops counting toward the plan's live apps. Archived sites still count toward the plan's total of sites and its storage, which only deleting frees. Suggest archiving for sites nobody uses.

`archived: false` brings a site back. When that would pass a plan limit, the site used least recently is archived in its place: tell the person which one from `archived_in_its_place`, which is null when none moved or when the one that moved is a site they cannot open. Artifen also archives sites nobody visited or changed for a while, and `pinned: true` keeps a site out of that when an owner or editor of its workspace pins it. A plan pins fewer live apps than it runs, so a refused pin means unpinning another first.

## With A Shell

When you can run commands, `npx artifen` sends local files and whole folders, and `git clone` and `git push` work on a site's source, where each push makes a version and builds it. Read [the shell reference](references/shell.md) before using either.

## When Something Does Not Fit

Tell the person what matters to them, such as the address, who can open it, and what changed, in plain words. File names, settings, and terms from these references are for your code, not for them, unless they ask.

When a request runs into a limit, an allowance, or something only the app does, say so in a sentence and offer the closest way that works: a plain static version, the work split into smaller pieces or batches, the heavy part run somewhere else that the site calls, the plan that allows it, or the step they take in the app.

Work within the limits rather than around them: never spread one site over several, make extra sites or accounts, or retry in a loop to get past an allowance or a refusal. Plan allowances change, so read them from results and from an error's `details`, never from memory. At a limit nothing is lost, and things are removed only with the person's agreement. Read [the limits](references/limits.md) when an allowance refuses a call, when a site's pages fail in use, and before building anything long-running.

## Errors

Every refusal is `{ "error": { "code", "message", "hint", "details" } }`, and a refused call changed nothing. A failed build is not a refusal: its change is saved, as Every Change Builds says. Follow the hint, read `details` for the field or limit involved, and read [error recovery](references/errors.md) for what to do next on any code.
