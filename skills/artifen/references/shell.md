# With A Shell

When you can run commands, the `artifen` command line sends files from disk, whole folders included, and Git works on a site's source. Both act as the person, under the same rules and errors as the tools.

## Sign In

`npx artifen` needs Node.js 22.13 or later. Run it as `npx --yes artifen`, so npx never stops to ask before installing it. Its sign-in is separate from the tools' connection. When a command answers exit code 4 or tells you to sign in, `npx artifen login` opens a browser on the same computer for the person to sign in, so run it only where they can finish it there, or ask them to run it, and wait. Never ask for a password or token.

## Commands

Add `--json` to get exactly one JSON document on standard output, the result or `{ "error": { "code", "message", "hint", "details" } }`. `npx --yes artifen <command> --help` gives each command's options.

| Task | Command |
| --- | --- |
| Create a site from a folder | `npx --yes artifen sites create ./my-site --workspace <workspace id> --title "My site"` |
| Create a site from the template | `npx --yes artifen sites create --template --workspace <workspace id> --title "My app"` |
| Write files or a folder into a site | `npx --yes artifen files write <site> ./dist` |
| Put one local file at a path | `npx --yes artifen files upload <site> images/photo.jpg ./photo.jpg` |
| Save a file exactly, binary included | `npx --yes artifen files read <site> <path> --out <file>` |
| Publish the preview | `npx --yes artifen sites publish <site>` |
| Make a Git credential | `npx --yes artifen git credential <site>` |
| Call a private site from here | `curl -H "Artifen-Site-Key: $(npx --yes artifen keys temporary <site>)" https://<site address>/api/...` |
| Make a key for a script elsewhere or a scheduled job | `npx --yes artifen keys create <site> "Nightly import"`, shown once |

A `<site>` is a site's 8-character ID or any of its addresses. `sites create` starts from exactly the files sent, and `sites create --template` from the template, where files given on top replace the template's files at the same paths. To build on the template locally, create it with no files, then clone it. A clone cannot install or type-check on its own, since `@artifen/site` comes with each build: push, and read the build's errors. Never add `@artifen/site` to `package.json`.

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

The last two lines keep Git asking the command line for a credential on later pulls and pushes, rather than any helper or password saved on the computer.

Each push makes a new version and builds it, the same as a tool change, and its result appears in `list_site_versions` and `read_site`'s `latest_build`. Pull before you push, since changes made through the tools or the app land as commits too, and merge them. Never force-push: a forced push replaces the site's history and whatever others changed. One push carries at most 64 MB and 50,000 objects. To send a larger history, push an older commit first, for example `git push origin HEAD~50:main`, then the rest.

A live app's stored file, a large file such as video it serves, goes in with `npx artifen files upload <site> <key> <file> --stored`, up to the plan's max upload size: it is sent in parts, running the command again carries an interrupted upload on, and it is served on the site's own address at `/__artifen/files/<key>`. Add `--preview` for the preview's stored files.
