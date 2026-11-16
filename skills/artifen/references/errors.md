# Error Recovery

Every refusal carries `code`, `message`, `hint`, and sometimes `details`, and a refused call changed nothing. The hint is written for every surface. This table adds what to do next.

| Code | What to do |
| --- | --- |
| `unauthorized` | The connection's sign-in expired or was revoked. Ask the person to sign in to Artifen again through this app's connection settings, or, for the command line, to run `npx artifen login`. When `details.reason` is `connection_revoked`, they disconnected this app under Connected agents in the Artifen app's settings and must allow it again there. Never ask for a password or token in chat |
| `forbidden` | The person's role on the site or workspace does not allow this, such as a Viewer changing files. Say which role it needs and that an Owner can grant it in the Artifen app |
| `app_only` | Sharing, roles, custom domains, GitHub sync, workspace members, and deleting a site for good are changed in the Artifen app. Tell the person where, from the hint |
| `not_found` | The ID does not exist or is not available to this person. List workspaces or sites, or read the site, for current IDs and paths. Never guess an ID |
| `address_not_found` | The HTTP API has no operation at that method and path. Check them against `/openapi.json` |
| `validation_failed` | An argument is malformed. `details` names the field and the rule. Fix that field and send again |
| `invalid_path` | Use a relative path with `/` separators and no `.` or `..` segments, such as `images/logo.png` |
| `reserved_path` | `__artifen/` and `.git` folders belong to Artifen. Store the file elsewhere |
| `read_required` | Read the file in full, every page of it, then replace it, or make an exact edit instead. `details.paths` lists every file of the call that needs a read |
| `conflict` | The file changed after your read. Read it again and reapply your change to what is there now. Never resend the version built from the old read |
| `edit_match_not_found` | `old_string` is not in the file, often because it changed. Read the file again and copy the text exactly from it |
| `edit_match_ambiguous` | `old_string` appears more than once. Include more of the surrounding text so it matches once |
| `edit_unsupported` | Exact edits apply to text files only. Replace the file whole after reading it |
| `patch_invalid` | A hunk does not match the file's current content, or the patch is malformed. Read the file the details name and rebuild that hunk from it |
| `patch_deletes_files` | Patches cannot delete. Delete those paths with `delete_site_files`, then send the patch without its `*** Delete File:` hunks |
| `file_too_large` | The file is over the per-file limit in `details`. Make it smaller, for example by compressing an image, and send it again |
| `payload_too_large` | The call carries too much. Send fewer files per call, or send large files as `source_url` or through an upload link |
| `quota_exceeded` | A plan allowance is used up, named in `details.allowance`. Tell the person what the message says. Remove things only with their agreement. The workspace's payer changes the plan in the Artifen app |
| `limit_reached` | A limit that is the same on every plan, named in `details.limit`. A new account's limits lift on the date in `details`. Otherwise make room or split the work |
| `rate_limited` | Wait `details.retry_after_ms`, then try once more. Combine changes into fewer calls |
| `try_again` | Something was still finishing. Try again in a moment. After a failed `create_site`, `details.site_created` says whether a site exists, and when it is absent, list the workspace's sites before creating again |
| `site_trashed` | The site is in Trash. Restore it with `restore_site` if the person wants to work on it again |
| `site_suspended` | The site was suspended. Nothing an agent can fix. Its owner sees why in the site's settings in the Artifen app |
| `nothing_to_publish` | A private site's changes are already live, and a shared site's preview already matches live. Nothing to do |
| `build_failed` | Publishing stopped because the preview's version did not build. Fix the errors in `details`, let the change build, then publish again. Live keeps serving what it served |
| `query_not_read_only` | Send one `SELECT`, `WITH`, or `EXPLAIN` statement. Data changes belong in the site's code and migrations |
| `query_failed` | The SQL did not run. Fix what the message names, such as a table or column, and run it again. Read `migrations/` for the schema |
| `secret_name_invalid` | Use capital letters, digits, and underscores, starting with a letter, such as `MAPS_API_KEY`. Names starting with `ARTIFEN_` are Artifen's |
| `upload_invalid` | The upload link or staged file was used, expired, or belongs to another site or person. Make a new one and use it once |
| `transfer_unsupported` | Nothing in the call carries this file's bytes. Send text as `content`, a public file as `source_url`, or make an upload link |
| `source_unreachable` | Artifen could not fetch `source_url`. Check that the URL is public and answers with the file itself, not a page about it |
| `joining_closed` | Joining a site by code has closed. The person who showed the code opens it again in the Artifen app |
| `account_closed` | This account was closed. The person signs in with another account |
| `account_suspended` | This account is suspended. The person writes to the address in the hint |
| `seat_suspended` | The workspace has more members than its plan admits, so this person's seat is paused. The workspace's Owner frees a seat or changes the plan |
| `not_available_yet` | This part of Artifen is not available yet, and nothing changed. Say so and carry on without it |
| `internal` | Something failed on Artifen's side. Try once more. After a failed `create_site`, list the workspace's sites first, since the site may exist |

## Build Errors

A failed build is not a refusal. The change is saved as a new version, the address keeps serving its last working version, and `build.errors` lists each problem with its `step`:

| Step | Usual cause |
| --- | --- |
| `install` | A package name or version in `package.json` that does not exist |
| `typegen` | `app/routes.ts` names a file that is missing, or cannot be read |
| `typecheck` | A type error, often a `./+types/<module>` import that does not match the route's file name, or a secret read from `env` without its name declared on `Cloudflare.Env` |
| `lint` | A lint rule, named in `rule` |
| `routes` | A module in `app/routes` that `app/routes.ts` does not list |
| `imports` | An import from `react-router-dom` or `@remix-run/*`. Import from `react-router` |
| `migrations` | A migration that does not apply after the ones before it, or that uses a statement migrations cannot run |
| `build` | Anything else the build stopped on, including bindings in `wrangler.jsonc` a site does not get, and, for a site with no `package.json`, a file only a build can serve |

`read_site_logs` with `source: "build"` and the `build_id` gives the whole log.
