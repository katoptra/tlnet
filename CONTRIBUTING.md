# Contributing

Pull requests are welcome, especially ones that make the pipeline smaller.

## Ground rules

- All logic lives in `Taskfile.yml` and in [katoptra/lib](https://github.com/katoptra/lib),
  whose toolbox and rsync engine it includes at `v1`. lib's README is the contract: the
  verbs, the vars, the image and its tools, the workflows and how they are pinned. A change
  to how bytes move or how the tree is verified belongs in the engine, where every mirror
  gets it; the filter that selects the subtree, the landing page and its row of the report
  belong here. Extension is a hook, never a copy of an engine verb.
- The mirror must stay inside the R2 free tier (10 GB, 1M Class A ops a month), and the
  pipeline refuses to run past 10 GB upstream. If a change adds storage or upload
  operations, say by how much in the PR.
- Objects stay under `systems/texlive/tlnet/`; every user's `tlmgr` config carries that
  path.

## Checking a change

```sh
task check                # render every command inside the image, diff it against render.txt
task run -- task list     # list the subtree through FILTER, no credentials; read .run/upstream.txt
```

`task run` pulls the image on first use and mounts the repo at `/work`. A change to what
the pipeline executes is a diff in `render.txt`: run `task render-update`, commit the
result with the change, and that diff is the review. The `check` workflow makes the same
comparison on every pull request.

The engine's own checks, the signed `prepare` and `verify` included, run in a checkout of
lib: `cd lib/examples/rsync && task run -- task offline`.

Uploads need real R2 credentials and have no mock; test them on your own fork (see "Want
your own?" in the README) with `BUCKET` in `Taskfile.yml` pointed at a scratch bucket.

## Commits

`<type>(<scope>): <summary>` in the imperative, under 75 characters. Types: feat, fix,
refactor, docs, test, chore, ci. One PR per change; PRs are squash merged.
