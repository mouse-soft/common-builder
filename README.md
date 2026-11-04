# common-builder

Manually dispatched GitHub-hosted builds for private projects, with private
release delivery. Builds run only on GitHub-hosted runners
(`ubuntu-24.04`, `windows-2022`, `macos-14`). No self-hosted runners.

## How it works

1. A maintainer (or an authorized agent) starts the `build` workflow manually
   from this repository, giving a registered project name, the exact
   40-character source commit SHA to build, and an output label.
2. The workflow reads the project registry (`projects.json`, in this public
   repository): source repository, build recipe path inside the source, and
   the return-branch prefix. Unregistered projects cannot be built.
3. The runner fetches the private source at the exact SHA over SSH, using a
   repository-scoped deploy key stored as an environment secret
   (`private-build` environment, secret `SOURCE_SSH_KEY`). The key works for
   that one source repository only.
4. The private build recipe runs with all of its stdout/stderr redirected to
   a file inside the runner's temporary directory. Nothing the compiler or
   the recipe prints reaches the public workflow log. On failure the public
   log shows a fixed message only.
5. Results return through git, not through GitHub Actions artifacts: the
   packaged zip (and, for diagnosis, the withheld build log) is pushed to a
   short-lived branch `common-builder-output/<project>/<run-id>-<os>` in the
   private source repository. A maintainer collects the zips from there,
   creates the private release, and deletes the branches.
6. The temporary source, outputs and SSH key are removed at the end of every
   job, success or failure.

## What never leaves the private side

- No source code, binaries, build logs, compiler errors or file paths from
  the private source are printed to public logs, job summaries, artifacts or
  caches. This workflow uploads no Actions artifacts and uses no caches.
- The workflow's own permissions are empty (`permissions: {}`). The only
  credential in the job is the deploy key, and it is never passed to the
  build recipe: fetching and returning happen in separate steps around the
  recipe run. (Steps share one runner, so this is separation of duties, not
  a sandbox. Only build registered projects whose recipes are trusted.)

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

## Registered projects

See `projects.json`. Adding a project means adding its entry here and
registering a repository-scoped deploy key (with push access, used only for
the return branches) as the `SOURCE_SSH_KEY` secret of the `private-build`
environment in this repository.

## First project

`obs-weather-plus` — `mouse-soft/obs-weather-plus`, recipe
`runtime/native/build-ci.py`. Builds the OBS Weather Plus native plugin and
its bundled Python renderer runtime for Linux, Windows and macOS.
