# common-builder

Manually dispatched GitHub-hosted builds for private projects, with private
release delivery. Builds run only on GitHub-hosted runners
(`ubuntu-24.04`, `windows-2022`, `macos-14`). No self-hosted runners.
This repository is public and contains **only generic build logic** — no
project names, source locations, product details, artifacts, detailed
logs, or secrets.

## How it works

1. A maintainer (or an authorized agent) starts the `build` workflow
   manually, giving one input: an opaque build-environment code. The code
   selects a GitHub environment in this repository. Nothing else is taken
   as an input: in particular the target commit is never an input, because
   workflow inputs and environment values are printed into the public
   workflow log.
2. Each project's configuration — source SSH URL, build recipe path
   inside the source, target ref (branch), and return SSH URL and branch
   prefix — is stored as a single secret (`BUILD_CONFIG_JSON`) in that
   project's environment, together with its own two deploy keys
   (`SOURCE_SSH_KEY`, read-only on the one source repository, and
   `RETURN_SSH_KEY`, write-only on the shared private drop repository).
   Projects never share a source key, and one project's key material is
   never overwritten by another's.
3. The workflow resolves the configuration into files inside the runner's
   temporary directory (never into workflow/job/step environment
   variables), then fetches the private source at the head of the
   configured ref over SSH. The exact resolved commit SHA is recorded
   only in the private build log. The fetch key file is deleted
   immediately after the fetch, before the build recipe runs, so no key
   file exists on the runner while project code executes.
4. The private build recipe runs with all of its stdout/stderr redirected
   to the private build log inside the runner's temporary directory.
   Nothing the compiler or the recipe prints reaches the public workflow
   log. Every `git`/`ssh` operation in the workflow is likewise redirected
   to private log files, because tool error messages can contain
   repository URLs, file names, and remote messages. On failure the
   public log shows a fixed message only.
5. Results return through git, not through GitHub Actions artifacts: the
   packaged zip (and, for diagnosis, the withheld build log) is pushed to
   a short-lived branch `<branch-prefix>/<run-id>-<platform>` in a
   dedicated private drop repository, using the project's return key.
   Zips over ~95 MB are split into ~90 MB parts (`<zip>.partaa`,
   `<zip>.partab`, …), because GitHub rejects files over 100 MB in a
   push; reassemble with `cat <zip>.part* > <zip>`. A maintainer collects
   the zips from there, creates the private release in the source
   repository, and deletes the branches.
6. The temporary source, configuration, outputs, and SSH key files are
   removed at the end of every job, success or failure.

## What never leaves the private side

- No project names, source or drop repository locations, commit SHAs,
  recipe paths, source code, binaries, build logs, compiler errors, file
  paths from the private source, artifact file names, or buyer/product
  details are printed to public logs, job summaries, artifacts or caches,
  and none are stored in this repository (including its registry — there
  is none). This workflow uploads no Actions artifacts and uses no
  caches.
- The workflow's own permissions are empty (`permissions: {}`). The only
  credentials in a job are that project's two repository-scoped deploy
  keys. Neither key can reach any other repository, the account, or
  billing, and each can be revoked independently on its repository's
  deploy-key settings. The keys are never passed to the build recipe:
  fetching and returning happen in separate steps around the recipe run,
  and a key file exists only during its own step. (Steps share one
  runner, so this is separation of duties, not a sandbox. Only build
  projects whose recipes are trusted.)

## Recipe contract

A registered project provides a recipe script inside its private source.
The workflow calls it as:

```
python <recipe> --platform <linux|windows|macos> --arch <x86_64|arm64> --out <dir>
```

The recipe must:

- build and package the project for the current runner OS,
- write exactly one distributable `.zip` into `<dir>`,
- exit non-zero on failure,
- not require network access to anything except public package/dependency
  sources, and never print secrets (its output is withheld anyway).

## Registering a project

Registration is entirely private-side:

1. Choose an opaque environment code (it is the only dispatch input, and
   it is visible in the public run list, so it must not identify the
   product).
2. Create a GitHub environment with that name in this repository and set
   three secrets in it: `BUILD_CONFIG_JSON` (source SSH URL, recipe path,
   target ref, return SSH URL, branch prefix), `SOURCE_SSH_KEY`
   (a new read-only deploy key registered on the source repository), and
   `RETURN_SSH_KEY` (a write deploy key registered on the drop
   repository). Issuing new keys is a persistent access change and needs
   the repository owner's approval each time.
3. Add the recipe to the private source, following the contract above.
