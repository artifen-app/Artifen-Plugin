# Saving Data From A Page

A plain page can save and read data, such as votes, check-offs, notes, or sign-ups, through Artifen's data store, with no code of its own on the server. It stays a page, not a live app. Keep entries in the store, never in the page's own files, where your next edit would overwrite them. When the site needs more than saving and reading entries under these rules, such as totals worked out on the server, it is a live app instead.

## Rules File

`artifen.data.json` at the site's root names each collection and who may read and write it. The build checks it and fails with the problem named when it is invalid, and it is never served.

```json
{
  "collections": {
    "votes": { "read": "everyone", "write": "everyone" },
    "notes": { "read": "signed_in", "write": "signed_in", "scope": "person" }
  }
}
```

- `read` and `write` are each `everyone`, `signed_in` (anyone signed in, through the site's visitor sign-in or as a member), or a member role, `viewer`, `commenter`, `editor`, or `owner`, which admits that role and the ones above it.
- In a `site` collection, anyone allowed to write can change or remove any entry, and `signed_in` includes guests who only gave a name. For "anyone can send, only I can read", use `write: "everyone"` and `read: "owner"`.
- `scope` is `site` (the default), one set every allowed person shares, or `person`, a set of its own for each signed-in person, which never takes `everyone`.
- Collection names start with a lowercase letter and use lowercase letters, digits, `-`, and `_`, up to 40 characters, and a site has at most 20.

## In The Page

```html
<script src="/__artifen/data.js"></script>
<script type="module">
  const votes = artifen.data("votes");
  const { entries, cursor } = await votes.list({ limit: 50 });
  await votes.add({ choice: "tea" });
</script>
```

- `artifen.data("<collection>")` gives `list({ cursor, limit })`, which answers `{ entries: [{ id, value, updatedAt }], cursor }` (up to 100 at a time, pass `cursor` on for more), `get(id)` (the entry or null), `set(id, value)` (answers `{ id, updatedAt }`), `add(value)` (answers `{ id, updatedAt }`), and `remove(id)` (answers `{ deleted }`).
- A value is any JSON up to 16 KB. Saved data counts toward the plan's database allowance. An ID is letters, digits, `-`, and `_`, up to 64 characters.
- Entries come in order of their IDs, which is the order they were added when `add` made them. There are no live updates, and saves and removals count toward the plan's server requests while reads do not, but every call counts toward the per-minute request limit in the limits reference, so read again when the page opens or comes back into view, or at a slow interval, never every few seconds.
- `artifen.me()` answers `{ id, role }` for the person signed in, or null. `role` is a member's role on the site, or `visitor` for someone signed in through the site's visitor sign-in. `artifen.signIn()` starts the site's visitor sign-in. For a collection limited to a member role, such as `owner`, members sign in through a link to `/__artifen/member?return=/path` on a site open to anyone, and on a private site they already are.
- Saves are limited per visitor and per site in a short time, and every call meets the per-minute request limit, answering `rate_limited`: wait a minute and try again.
- A failed call throws an error with `code`, `status`, and `message`. Show its message to the person in the page.

A shared site's published version and its preview each have their own data, and a preview starts with a copy of the published data. A private site has only the published data.

## Reading, Clearing, And Limits

- `query_site_data` reads a page's saved data: for a site with no database of its own, the database is one table, `entries` (`collection`, `owner`, `id`, `value` as JSON text, `updated_at`), where `owner` is the person's ID on the site in a collection kept per person and empty otherwise.
- Your tools cannot change or clear entries. Someone who can change the site clears a whole collection in the Artifen app, on the site's Data tab, under "Saved by its pages", or the page itself has a delete or clear control.
- Collections anyone can write to, `everyone` or `signed_in`, hold 10 MB together on the published site and on its preview, and a save past that answers `limit_reached` until entries are removed. All of a site's saved data stops at 100 MB.
- While the plan's usage budget is used up, reads and deletes still work and only saves are refused. While a site is archived, every change is refused. The refusal's message says why.
