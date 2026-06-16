---
name: tea
description: Use the tea CLI to work with Gitea or Forgejo git servers (including self-hosted ones like git.altinkaya.com) — pull requests, issues, reviews, releases, repos, labels, and raw API calls. Use this whenever a repo's remote is NOT GitHub and you would otherwise reach for `gh`; tea is the Gitea/Forgejo equivalent of the GitHub CLI. Covers login/auth, the gh→tea command map, the non-obvious flag names (a PR/issue body is `--description`, not `--body`), and how to avoid interactive prompts that hang automation.
---

# tea — the Gitea / Forgejo CLI

`tea` is `gh` for Gitea and Forgejo servers. Forgejo is Gitea-API-compatible, so the same
`tea` works against a Forgejo host like `git.altinkaya.com`. Verified against **tea 0.14.1**.

Mental model: most commands act on the repo inferred from the **git remote in `$PWD`**, and
need a configured *login* whose URL matches that remote's host. Logins live in `~/.config/tea/`.

## Gotchas that break agents (read first)

These are where `gh` muscle memory and reasonable guesses go wrong:

- **Interactive prompts hang automation.** `tea pr create` with no `--title`, `tea login add`
  with no flags, `tea pr review`, and any command given `--comments` will drop into an
  interactive TUI and block forever. Always pass every field as a flag. For reviews, use the
  non-interactive `tea pr approve` / `tea pr reject`, not `tea pr review`.
- **The body flag is `--description` / `-d`, NOT `--body`.** And on `pr create`, `-b` is
  `--base` (the target branch), so `-b "my text"` silently sets the wrong thing. Use `-d`.
- **View an item by its bare index** — there is no `view`/`show` subcommand. `tea pr 42`,
  `tea issue 7`, `tea milestone "v2"`, `tea branch main` all show detail.
- **`-R` is a trap.** gh's `-R owner/repo` does NOT exist in tea. In tea, `--repo` / `-r`
  takes the slug *or* a local path; `--remote` / `-R` means "discover the login from this git
  *remote name*." gh `-R owner/repo` → tea `--repo owner/repo`.
- **Push before you PR.** tea assumes your local branch is already published on the remote.
  `git push -u origin <branch>` first, or `tea pr create` won't find your changes.
- **Context flags go *after* the subcommand.** The only global flags are `--debug`,
  `--help`, `--version`. `--repo`, `--login`, `--remote`, `--output` attach to the subcommand:
  `tea pr list --repo owner/name --login altinkaya`.
- **Machine-readable output:** add `-o json` (formats: `simple table csv tsv yaml json`).
- **Pick a server with `--login <name>`** when more than one is configured; `tea whoami`
  confirms who/where you are.

## Auth (one-time, non-interactive)

```bash
# Token: Gitea/Forgejo → Settings → Applications → Generate New Token
tea login add --name altinkaya --url https://git.altinkaya.com --token <TOKEN>

tea login list            # list configured servers
tea login default altinkaya
tea logout altinkaya
tea whoami                # who am I on the active login?
```

Env vars work too: `GITEA_SERVER_URL`, `GITEA_SERVER_TOKEN`. Restrict a token's reach with
`--scopes`. Basic auth (`--user` / `--password`) auto-creates a token if you have no PAT.

## Pull request workflow

The everyday loop (matches a "branch → push → land via PR" workflow):

```bash
git switch -c feature && git push -u origin feature        # publish FIRST

tea pr create --title "Add X" --description "Why X" --base main
#   head defaults to the current branch; --base defaults to the repo default branch
#   cross-fork:  --head someuser:their-branch
#   extras:      --labels bug,ui   --assignees alice   --milestone v2

tea pr list                       # open PRs;  --state all|closed for the rest
tea pr 42                         # view PR #42 in detail (bare index, no "view")
tea pr checkout 42                # check it out locally ( -b creates the branch)
tea pr approve 42 "LGTM"          # non-interactive approve  (alias: lgtm, a)
tea pr reject 42 "needs changes"  # request changes  (same <index> [comment] shape)
tea comment 42 "deploying now"    # comment on PR or issue #42
tea pr merge 42 --style squash    # styles: merge | rebase | squash | rebase-merge
tea pr close 42                   # or: reopen
tea pr clean 42                   # delete local+remote feature branch of a closed PR
```

Avoid `tea pr review 42` in automation — it's an interactive TUI.

## Issues

```bash
tea issue create --title "Bug" --description "repro steps" --labels bug --assignees bob
tea issue list                    # --state all  --keyword "crash"  --labels bug  --assignee me
tea issue 7                       # view issue #7
tea issue close 7                 # or: reopen 7   (both take multiple indices)
tea issue edit 7 --add-labels p1 --milestone v2
tea comment 7 "fixed in #42"
```

## Repos

```bash
tea clone owner/repo [dir]        # top-level helper; NOT "tea repo clone"
#   slug is flexible: owner/repo · repo · git.altinkaya.com/owner/repo · full ssh/https URL
#   a host in the slug overrides --login;  --depth N for shallow
tea repo create --name newrepo --private --init
tea repo fork                     # fork the current repo  (--owner to place it elsewhere)
tea repos list                    # --watched  --starred  --owner someorg
tea repo search "kw"
tea repo owner/repo               # view repo details
```

## Releases

```bash
tea release create v1.2.0 --title "v1.2.0" --note "changelog..." \
    --asset ./dist/app.tar.gz          # --asset repeatable; --note-file FILE; --draft; --prerelease
tea release list
```

## tea api — the escape hatch (mirrors `gh api`)

For anything tea doesn't wrap. Auto-prefixes `/api/v1/`; fills `{owner}`/`{repo}` from `$PWD`.

```bash
tea api repos/{owner}/{repo}/issues                 # GET by default
tea api -X POST repos/{owner}/{repo}/issues -f title="Bug" -f body="..."
tea api -X PATCH repos/owner/name -F private=true   # -F = typed (num/bool/null), -f = string
tea api '/repos/{owner}/{repo}/issues?state=open'   # quote URLs with ? or &
```

## Everything else (run `tea <cmd> --help` for flags)

`tea notifications` (alias `n`) · `tea labels` · `tea milestones` · `tea org` ·
`tea branches` · `tea times` · `tea actions` (runs/secrets/variables/workflows) ·
`tea webhooks` · `tea ssh-keys` · `tea open` (open in browser) · `tea admin`.

This skill covers the common 90%. `tea <cmd> --help` is the authoritative, always-current
source for the long tail of flags — prefer it over guessing.

## gh → tea cheat sheet

| GitHub `gh` | Gitea/Forgejo `tea` | Note |
|---|---|---|
| `gh auth login` | `tea login add --name N --url URL --token T` | no-flag form is interactive |
| `gh auth status` | `tea login list` / `tea whoami` | |
| `gh repo clone owner/repo` | `tea clone owner/repo` | top-level, not `tea repo clone` |
| `gh repo create` | `tea repo create --name N` | |
| `gh repo fork` | `tea repo fork` | |
| `gh repo list` | `tea repos list` | |
| `gh repo view owner/repo` | `tea repo owner/repo` | |
| `gh pr create -t T -b BODY -B BASE` | `tea pr create -t T -d BODY -b BASE` | **body is `-d`; `-b` = base** |
| `gh pr list` | `tea pr list` | `--state all` for closed |
| `gh pr view 42` | `tea pr 42` | bare index |
| `gh pr checkout 42` | `tea pr checkout 42` | |
| `gh pr merge 42 --squash` | `tea pr merge 42 --style squash` | merge/rebase/squash/rebase-merge |
| `gh pr review --approve` | `tea pr approve 42` | non-interactive |
| `gh pr review --request-changes` | `tea pr reject 42 "why"` | non-interactive |
| `gh pr comment 42 -b "x"` | `tea comment 42 "x"` | same cmd for issues |
| `gh pr close 42` | `tea pr close 42` | |
| `gh issue create` | `tea issue create -t T -d BODY` | body is `-d` |
| `gh issue list` | `tea issue list` | |
| `gh issue view 7` | `tea issue 7` | |
| `gh issue close 7` | `tea issue close 7` | |
| `gh release create v1 -t T -n NOTES` | `tea release create v1 -t T -n NOTES` | |
| `gh api PATH` | `tea api PATH` | auto-prefixes `/api/v1/` |
| `gh browse` | `tea open` | |
| `gh run list` | `tea actions runs list` | |
| `gh -R owner/repo <cmd>` | `tea <cmd> --repo owner/repo` | **not `-R`** |

gh-only (no `tea pr create` equivalent): `--draft`, `--web`, `--fill`. Open the PR normally,
then mark it draft in the UI or via `tea api` if needed.

## Verify it works

```bash
tea --version                 # expect 0.14.x — flags here match that line
tea whoami                    # a login is configured and reachable
tea pr list -o json           # (inside a repo) structured output works
```
