# Editing The artifen Skill

`SKILL.md` and its references are what agents read while they work. `README.md` is the short version for people.

## File Roles

| File | Role |
| --- | --- |
| `SKILL.md` | Always-needed guidance: when to build on Artifen, creating a site, sharing, changing files, builds, transfer, preview and publish, history, logs, data, secrets, comments, Trash, errors |
| `references/site-stack.md` | Writing code for a site that starts from the template: what each build checks, the layout, and everything `@artifen/site` provides |
| `references/errors.md` | The next move for every error code and every build step |
| `references/shell.md` | The command line and Git, for agents that can run commands |
| `references/limits.md` | One request's CPU time, what to build outside a site, and how plan allowances show up |

## Editing

- Teach Artifen's public contract as built: the tools' own schemas, their results, and their error codes. Tool descriptions say what each tool does, and the skill says when, why, and how to use it.
- Teach only what an agent would get wrong without it.
- Never copy a plan allowance or limit into the skill. Point at where results and errors report it.
- Name no provider Artifen runs on. Import paths and file names a site's own code must use are the exception.
- Quote every frontmatter string value, and bump `metadata.version` with each change people receive.
- No em dashes or en dashes, and no semicolons joining sentences.
- The skill passes the skill-forge validator.
