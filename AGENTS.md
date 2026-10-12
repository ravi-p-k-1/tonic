# AGENTS.md — tonic

Notes for an AI coding agent working in this repo. See [README.md](README.md)
for what this Worker does and [PLAN.md](PLAN.md) for the full design,
including the downstream (separate-repo) processing side.

## What this is

A single Cloudflare Worker, no build step, no framework. `src/index.js` is
the entry point; everything else in `src/` is a plain ES module it imports.
Keep it that way — don't introduce a bundler, a router library, or a
framework for a four-file Worker.

## Before changing `src/validate.js`

The exact field values TranscripTonic sends (`meetingSoftware`, etc.) were
determined by reading the extension's own source and, once, by a real
rejected payload in production — not by guessing from documentation
wording. If you're tempted to "fix" a literal string here, verify it
against a real payload or the extension's source
(`vivek-nexus/transcriptonic` on GitHub) first. A previous version of this
file shipped with `meetingSoftware` checked against the wrong casing
(`"google_meet"` instead of the real `"Google Meet"`) for exactly this
reason — it looked right by analogy to an unrelated internal id elsewhere
in the extension's code, and broke every real meeting until caught.

## Testing

```bash
npm test
```

Runs plain Node (`node --test`), not Wrangler or Miniflare — `src/github.js`'s
tests mock the global `fetch`, they don't hit a real GitHub API. If you add
a new test file, add it explicitly to the `test` script in `package.json`
rather than relying on directory globbing (`node --test test/` has shown
path-resolution issues in at least one local Windows/Git Bash setup during
this project's development — explicit file lists sidestep it entirely).

Before committing a change to `src/`, run `npm test` and confirm it's
green. If you change `src/validate.js` or `src/github.js`, check whether
`test/worker.test.js` / `test/github.test.js` need a matching update —
they're not auto-generated.

## Secrets

Never print, log, or commit a real `WEBHOOK_SECRET` or `ARCHIVIST_TOKEN`.
`.dev.vars` is git-ignored for exactly this reason — if you need to
demonstrate a command that uses one, use a placeholder like
`<your-webhook-secret>`, not a real value, even if the user has shared one
with you in conversation.

If you're setting a secret via `gh secret set`, prefer letting the human
run it themselves (it prompts for the value without it ever appearing in
your own output) over passing `--body <value>` where you'd have to handle
the raw secret text yourself.

## Testing against the live Worker (not just unit tests)

If you replay `test/fixtures/sample-meeting.json` against a **deployed**
Worker (not local `wrangler dev`) to verify end-to-end behavior, it
creates a real branch and PR on the real target repo (`ARCHIVIST_REPO`).
Two things to watch for:

- The fixture's `meetingStartTimestamp` must not land on today's Pacific
  date, or it'll collide with the duplicate-date check against a real
  meeting that might land the same day. Keep it set to a date safely in
  the past.
- Clean up afterward: close the PR and delete the branch
  (`gh pr close <n> --repo <owner>/<repo> --delete-branch`) so test data
  doesn't linger on a real repo.

## Deploying

Any `git push` to `main` triggers `.github/workflows/deploy-transcript-intake.yml`
automatically — there's no separate manual deploy step to remember, and no
path filter to worry about since this whole repo is one Worker.

If this repo is checked out as a git submodule inside a larger project
(e.g. `cloudroot`), its own `.github/workflows/` only runs as Actions on
*this* repo, not the parent — the parent project needs its own workflow if
it wants to trigger a deploy from a submodule bump.

## Scope boundary

This repo only covers "receive the webhook, validate it, open a PR."
Everything downstream of the PR (extracting action items, committing
results back, archiving) is a separate repo's responsibility — don't add
that logic here even if it seems convenient. See PLAN.md's "archivist1
changes" section for what that side looks like, if you need to understand
the full picture.
