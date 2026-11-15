---
name: "artifen"
description: "Use when building, publishing, updating, or fixing a site, web app, page, game, dashboard, or slideshow on Artifen (artifen.app, artifen.site), and whenever a task calls Artifen's tools such as create_site, write_site_file, or publish_site, or the `npx artifen` CLI. Also use whenever the person asks for an artifact, page, app, or slideshow they can share, send, or open on another device while Artifen is connected, even without naming Artifen. Teaches when to build on Artifen, how to start, write, and change a site from plain files to a full-stack app, publish it, and recover from every error."
license: "Apache-2.0"
metadata:
  author: "Leeor Nahum"
  version: "0.1.0"
---

# Artifen

Artifen puts what you build online as a site with its own address on `artifen.site`, from one HTML page to a full-stack app with a database, sign-in, files, email, AI, and payments. The tools, the `npx artifen` command line, and the HTTP API reach the same sites under the same rules.

## Build Here When The Person Wants To Share

When Artifen is connected and the person asks for a page, slideshow, game, dashboard, app, or artifact they can share, send, keep, or open on a phone, build it as an Artifen site instead of in the host's own artifact or canvas view, since a site has an address the person can give to others and keeps working after the chat ends. Say that you are putting it on Artifen.

When the tools are missing and you cannot run `npx artifen`, tell the person they can connect Artifen to this app by following http://localhost:3000/connect.

## Create A Site

1. Call `list_workspaces` and pick one where `can_create_sites` is true. When several qualify and the person has not said which, ask, since the workspace owns the site and decides who else sees it.
2. Choose where it starts, `start_from`:

   | The site | `start_from` |
   | --- | --- |
   | Runs entirely in the browser and saves nothing other visitors see: a page, a slideshow, a game, a calculator | `files`: plain HTML, CSS, and JavaScript, served exactly as sent, with no build |
   | Saves data that visitors share, signs people in, runs code on the server, sends email, uses AI, takes payments, runs on a schedule, or serves an API or MCP endpoint | `template`: Artifen's starting app, React Router with server rendering, Tailwind, a database, and live updates |

3. Call `create_site`.
   - With `files`, send `index.html` and every file the site needs in one call. Keep `package.json` out of the root, since it turns the site into an app that must build, and keep out TypeScript, JSX, Sass, and other files only a build can serve, which fail the build. There is no fallback page, so a client-side router needs hash routes or one HTML file per route. A `404.html` becomes the not-found page.
   - With `template`, read the new site's `AGENTS.md`, `app/routes.ts`, and the sample pages before changing them, since files you send with `create_site` replace the template's files at the same paths. Read [the site stack](references/site-stack.md) before writing any code for it.
4. Check `change.build` in the result (see Every Change Builds), then give the person `site.url`.

Each `create_site` call makes another site, so after a failure that may have landed, call `list_sites` before trying again.

## Who Can Open It

A new site opens only for the person and anyone invited to it. Sharing, roles, custom domains, GitHub sync, billing, and deleting a site for good are changed in the Artifen app, never by a tool. A tool call that reaches one of them answers `app_only`.

- Tell the person who can open the site now, from `site.access_level` (`private` is invited people only, `link` is anyone with the link, `public` is anyone, and found by search engines).
- Tell them where to change it: the site's page in the Artifen app, `http://localhost:3001/sites/<site id>`, under Share. They can invite people by email, open it to anyone with the link, make it public, or let a room of people join with a QR code.
- When you operate a browser the person is signed in to and they ask you to share the site, you may do it through that same Share dialog.

For "make me a slideshow I can share": build it, give the link, and say it opens only for them until they choose Share, then Anyone with the link, in the app.

## Change A Site

To find a site the person means, call `list_sites` with words from it as `query`. Then start with `read_site` and no `path`: it returns the site's addresses, your capabilities, the published and preview versions, the latest build, secret names, and the file list.

- **Exact edits first.** `edit_site_file` replaces `old_string`, which must appear exactly once in the file's current content. Copy it from a fresh read, never from memory.
- **Several files together.** `apply_site_patch` adds and changes files as one version, all or nothing. It never deletes. Use `delete_site_files` with exact paths for that.
- **Whole files need a full read.** `write_site_file` replaces an existing file only after you read all of it, following `next_offset` until it is null, and refuses one that changed since (`conflict`). A read made in another conversation through the same app can count as yours, so read the file again just before replacing it.
- **One change per intent.** Put every file of one change in one call. Each change makes a version and a build, and open pages show it once the build succeeds.

## Every Change Builds

Every change returns `build` with a `status`:

- `succeeded`: `url` in the result serves the change.
- `failed`: the change is saved as a version, and the address keeps serving its last working version. Each entry in `build.errors` names the `step` that failed, the `file`, `line`, and `message`. Fix them and send the fix as the next change.
- `queued` or `running`: not finished. Check `latest_build` from `read_site`, or `read_site_logs` with `source: "build"`, before telling the person it works.
- `superseded`: a newer change reached the address first.

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

While a site is private, every change goes live at `site.url` at once. Once the person shares it beyond invited people, changes land in its preview instead, at `site.preview_url`, which shows a Preview badge and has its own copy of the database, test secrets, email that reaches only the owner, and test payments. `publish_site` sends the preview live and applies its database migrations to the live data, and if anything fails, the address keeps serving what it served before.

After changing a shared site, give the person `preview_url` to check, and publish when they want the change live. `nothing_to_publish` means the site is private, so its changes are already live, or the preview matches what is live.

## History

Every change is a version. `list_site_versions` lists them newest first, with bookmarks and which version each address serves. `read_site` with `version_id` and `path` reads a file as it was. `restore_site_version` puts the files back as they were at a version, as a new version, so nothing in between is lost. The database is not rolled back.

## Logs, Data, Secrets, And Comments

- `read_site_logs` reads a build's log (`source: "build"`) or what the site's server code logged (`source: "server"`), for the live site or its preview (`slot`).
- `query_site_data` runs one read-only SQL query on the live database or the preview's copy. Data changes go through the site's own code and migrations.
- `set_site_secret` stores a key the site's server code reads by name, separately for `live` and `preview`, which should hold test keys. When the person pastes a key into the chat, store it as they ask. Never repeat its value in a reply or write it into a file. `delete_site_secret` removes one.
- `list_site_comments` reads what people left on the site's pages, and `reply_to_site_comment` answers one as the person.

## Delete And Restore

Call `delete_site` only when the person asks to delete the site itself. It moves the site to Trash and its address stops at once. Tell the person the date in `purge_after`, when it is deleted for good, and that it can come back until then. `restore_site` brings it back with its files, versions, data, and address, and so does Trash in the Artifen app.

## With A Shell

When you can run commands, `npx artifen` sends local files and whole folders, and `git clone` and `git push` work on a site's source, where each push makes a version and builds it. Read [the shell reference](references/shell.md) before using either.

## Errors

Every refusal is `{ "error": { "code", "message", "hint", "details" } }`, and a refused call changed nothing. A failed build is not a refusal: its change is saved, as Every Change Builds says. Follow the hint, read `details` for the field or limit involved, and read [error recovery](references/errors.md) for what to do next on any code.

Plan allowances and limits change, so read them from results and from an error's `details`, never from memory. At a limit nothing is lost. Remove things only with the person's agreement.
