# tlnet

A daily mirror of `CTAN/systems/texlive/tlnet`, the directory `tlmgr` installs and updates
from, on Cloudflare R2, served at `https://tlnet.katoptra.org/systems/texlive/tlnet/`.
`README.md` says what it mirrors, how to use it, how it works and how to fork it;
[katoptra/lib](https://github.com/katoptra/lib)'s README is the manual for everything the
mirrors share. This file is what a change must not break.

Nothing in this repo starts a run: an external scheduler dispatches `sync.yml` daily at
03:30 UTC, which is the hour `reconcile` keys on. `Taskfile.yml` and its comments are the
design of what is this mirror's own: the identity and the `FILTER` in root vars, `index`
(the landing page) and `report-mirror`. `aws.config` keeps every upload single-part.
`site/index.html` is the landing page, uploaded dated every run; it repeats the README's
"How to use", so a README edit there is usually a page edit too.

## Constraints

- Zero running cost: R2 free tier (10 GB-month, 1M Class A ops) and free GitHub Actions.
  `CEILING_GB: 10` makes `split` refuse a tree past it. Recompute any change that adds
  storage or uploads against the 6.8 GB baseline.
- No shell scripts. Logic lives in `Taskfile.yml` and, for everything shared with the
  other mirrors, in lib; a change to how bytes move or how the tree is verified goes to
  the engine, where every mirror gets it. Extension is a hook, never a copy.
- Excludes are `report-engine` and `report-mirror` on the toolbox include, `index` on the
  engine's. Never redefine a lib var. Inside an engine verb a root var shadows a
  command-line `KEY=value`, so `MAX_BATCHES` and `RECONCILE` stay out of the root vars and
  a run sets them: `task sync -- RECONCILE=true`.
- Objects stay under `systems/texlive/tlnet/`; every user's `tlmgr` config carries that
  path. `SOURCE` is CTAN's root and `FILTER` narrows the listing to the subtree, which is
  what keeps the prefix. `.state/` and `index.html` are the bucket's only other keys.

## Must knows

Each of these is a bug that has happened or a bill that would. Do not undo them.

- **No versioned containers.** `tlmgr` requests `archive/foo.tar.xz`; upstream stores
  `foo.r123.tar.xz` and symlinks to it. R2 has no symlinks, so the engine lists and fetches
  with `-L`, and `FILTER` excludes `*.r[0-9]*.tar.xz` and the `update-tlmgr-r*` updaters
  before its includes open the subtree. Storing both doubles storage and breaks the free
  tier. rsync takes the first rule that matches, so the order of `FILTER` is load-bearing.
- **The mirror is a list diff.** `list` takes dante's `rsync -rL --list-only` through
  `FILTER`, `diff` compares it with the state file the last run left in the bucket, and
  only the delta is fetched. The state line is `path TAB size TAB mtime`, so a revision
  bump that keeps a file's size is still caught by its mtime. A missing state is rebuilt
  from a bucket listing joined to upstream on size, never treated as empty: the bucket is
  the mirror and the state a cache of it. There is no seed.
- **The decision batch is last.** `split` puts `tlpkg/` after every container, and
  `verify` refuses that batch unless the tlpdb it carries is the one `prepare` verified at
  the start of the run, `texlive.tlpdb.xz` decompresses to it byte for byte, and every
  container it names is in the bucket after this run. `delete` waits for the run in which
  every batch has landed, so the live tlpdb never names a removed container.
- **Single-part uploads.** `aws.config` sets `multipart_threshold` above the largest file.
  The CLI default is 8 MB, and multipart costs three or more Class A ops per file.
- **`index.html` lives at the bucket root**, outside the subtree, uploaded with
  `Cache-Control: no-cache`, and `OWN: index.html` keeps `reconcile` from deleting it as an
  orphan. A Cloudflare Transform Rule rewrites `/` to `/index.html`; without it the root is
  a 404 and the mirror still works.
- **`reconcile` runs in the run that starts in hour 03 UTC**, which the 03:30 dispatch is,
  so the bucket is swept daily: the state is rebuilt from a listing, and every key neither
  upstream, `.state/` nor `OWN` holds is deleted. A run queued behind a long one can start
  in 04 and skip the day; `task sync -- RECONCILE=true` asks for it by hand.
- **Do not trust the job log for counts.** `report` counts from `.run/`, never the log.
  Judge completeness by `smoke`, never by counting.
- **A failed run is the only alert.** `split` fails before anything is fetched if upstream
  exceeds 10 GB; the mirror stays a day stale. healthchecks.io emails when a day passes
  without `ping`, which also catches a dispatcher that stops firing. Do not add
  notification dependencies or upstream-freshness monitoring (tlnet goes quiet for weeks
  before each release).
- `.xz` is not in Cloudflare's default cache list, so there is no stale-edge problem. If a
  cache-everything rule is ever added, the pipeline needs a purge step before `smoke`.

## Verification and security

`prepare` runs after `split` and before anything is fetched, and `verify` runs on every
batch before it is uploaded. A batch that fails stays local; the previous good copy stays
live.

1. `prepare` fetches `texlive.tlpdb.sha512`, its `.asc` and the keyring from the mirror
   being verified, checks the sha512 with `shasum`, then the signature with `gpgv`. The
   keyring is read as a plain file, so nothing is imported before it is authenticated.
2. Because the keyring came from the same mirror as the signature, the only real check is
   the pinned fingerprint `TL_KEY`: `VALIDSIG` must end in it. Rotate it only against
   https://www.tug.org/texlive/verify.html. `GOODSIG` is also required because gpgv reports
   an expired or revoked key as `VALIDSIG` with exit 0 (tlmgr accepts that; the mirror is
   stricter). The signing subkey expires 2027-07-13; upstream extends it yearly.
3. `verify` checks every signed sha512 in a batch (`texlive.tlpdb.sha512`, and at the tree
   root the `install-tl*.sha512` and `update-tlmgr-latest.*.sha512` files) the same way,
   and every container in the batch against the `containerchecksum`,
   `doccontainerchecksum` or `srccontainerchecksum` the verified tlpdb records (source
   containers are `<name>.source.tar.xz`). A container the tlpdb does not describe fails
   the batch.
4. In the decision batch, `texlive.tlpdb.xz`, the file `tlmgr` actually downloads, must
   decompress to the verified `texlive.tlpdb` byte for byte, and every container the tlpdb
   names must be in the bucket after this run.

After the batches, `smoke` reads `texlive.tlpdb.sha512` back through the public domain and
`cmp`s it with the verified copy, and sizes a sample of the run's uploads against the
listing. `tlmgr` repeats the signature check on the client.

Nothing else is signed upstream, so the rest is copied as-is: `install-tl`,
`install-tl-windows.bat`, `README.md` and `texlive.tlpdb.md5`, plus `tlpkg/TeXLive/`,
`tlpkg/installer/`, `tlpkg/tlperl/`, `tlpkg/tltcl/` and `tlpkg/translations/`. No TeX Live
tool fetches those from a repository URL; they serve installs run from a local copy of the
tree.

## Repository guardrails

- Actions are pinned to a full commit SHA with the version in a trailing comment (repo
  setting rejects tags). The allowlist is GitHub-owned plus `go-task/*`; lib's reusable
  workflows and action pass as same-org references.
- Dependabot groups Actions bumps weekly; these are the only routine commits.
- CodeQL default setup scans the workflows. Private vulnerability reporting is on and
  `SECURITY.md` links to it.
- PRs target `main` and are squash merged. `check / check`, the reusable check's name,
  must pass.
- Commits: `<type>(<scope>): <summary>`, imperative, under 75 characters.

## Verifying a change

Every check runs inside the toolbox image.

- `task check` renders every command of the pipeline inside the image and diffs it
  against `render.txt`; `task render-update` accepts a change. The `check` workflow does
  the same on every pull request.
- `task run -- task list` lists the subtree through `FILTER` with no credentials.
  `.run/upstream.txt` must hold only `systems/texlive/tlnet/` paths, none matching
  `\.r[0-9]+\.tar\.xz$`, none under `update-tlmgr-r`, and about 17,000 lines;
  `LIST_FLOOR` is half that.
- `task plan` runs the read-only half against the real bucket, through `op run`: the
  listing, the state, the delta and its batches, nothing uploaded.
- The engine's verbs, `diff`, `split`, `merge`, `retry` and the signed checks `prepare`
  and `verify`, are checked in lib: `cd ../lib/examples/rsync && task run -- task offline`.
- `publish`, `checkpoint`, `delete`, `rebuild` and `index` need credentials; a fork tests
  them with `BUCKET` in `Taskfile.yml` pointed at a scratch bucket and
  `task sync -- MAX_BATCHES=1 BATCH_GB=1`. A root var shadows the command line inside an
  engine verb, so `task sync -- BUCKET=x` changes nothing.
- Is the mirror fresh?
  `curl -sI https://tlnet.katoptra.org/systems/texlive/tlnet/tlpkg/texlive.tlpdb.sha512`
  and read `last-modified`.
