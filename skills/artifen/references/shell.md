# With A Shell

When you can run commands, the `artifen` command line sends files from disk, whole folders included, and Git works on a site's source. Both act as the person, under the same rules and errors as the tools.

## Sign In

`npx artifen` needs Node.js 22.13 or later. Run it as `npx --yes artifen`, so npx never stops to ask before installing it. Its sign-in is separate from the tools' connection. When a command answers exit code 4 or tells you to sign in, `npx artifen login` opens a browser on the same computer for the person to sign in, so run it only where they can finish it there, or ask them to run it, and wait. Never ask for a password or token.

## Commands

Add `--json` to get exactly one JSON document on standard output, the result or `{ "error": { "code", "message", "hint", "details" } }`. `npx artifen <command> --help` gives each command's options.

| Task | Command |
| --- | --- |
| Create a site from a folder | `npx artifen sites create ./my-site --workspace <workspace id> --title "My site"` |
| Write files or a folder into a site | `npx artifen files write <site> ./dist` |
| Put one local file at a path | `npx artifen files upload <site> images/photo.jpg ./photo.jpg` |
| Save a file exactly, binary included | `npx artifen files read <site> <path> --out <file>` |
| Publish the preview | `npx artifen sites publish <site>` |
| Make a Git credential | `npx artifen git credential <site>` |

A `<site>` is a site's 8-character ID or any of its addresses. `sites create` starts from the files sent. To start from the template, create the site with the `create_site` tool and `start_from: "template"`, then clone it.

Exit codes: 0 done, 1 failed (the error says why), 2 the command was not valid, 3 the change was saved and its build failed, 4 sign in first.

Errors carry the same codes as the tools, plus five the command line adds: `usage` (fix the command from `--help`), `unreachable` (check the network, then retry), `unexpected_response` (retry once, then report it), `local_file` (a local path is missing or unreadable), and `cancelled`.

## Sending Folders

A folder sends every file under it at its path inside that folder. `.git` and `node_modules` folders, `.env` files other than `.env.example`, and what the folder's `.gitignore` files leave out stay behind, and the summary lists them. A file whose bytes already match the site's copy is skipped. Replacing a file follows the same rule as the tools: read it first with `files read`, or pass `--overwrite` to replace files as they were when the command listed the site. When a send stops part way, its error names what landed and the command that sends the rest.

## Git

Every site is a Git repository, and its address is `clone_url` from `read_site`. Clone with the command line as Git's credential helper, which makes a short-lived credential whenever Git needs one:

```sh
git -c credential.helper= -c "credential.helper=!npx --yes artifen git credential <site>" clone <clone_url>
cd <folder>
git config credential.helper ""
git config --add credential.helper "!npx --yes artifen git credential <site>"
```

Each push makes a new version and builds it, the same as a tool change, and its result appears in `list_site_versions` and `read_site`'s `latest_build`. Pull before you push, since changes made through the tools or the app land as commits too. A push that would overwrite them is refused.
