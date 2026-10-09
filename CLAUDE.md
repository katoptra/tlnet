# tlnet

tlnet is a daily mirror of `CTAN/systems/texlive/tlnet` on Cloudflare R2, at
`https://tlnet.katoptra.org/systems/texlive/tlnet/`. `tlmgr` installs and updates TeX Live
from this directory. `README.md` tells the content of the mirror, how to use it, how it
operates and how to fork it. The README of [katoptra/lib](https://github.com/katoptra/lib) is
the manual for the parts that all the mirrors use. This file gives the rules that a change
must obey.

No file in this repository starts a run. An external scheduler dispatches `sync.yml` daily at
05:42 UTC. The hour of the run has no effect on `reconcile`.

## Constraints

- Where a change goes:
  - `Taskfile.yml` has these verbs: `index` (the landing page) and `report-mirror` (one row
    of the run summary). Make all other changes to verbs in lib: in the toolbox or in the
    rsync engine. Then each mirror that includes that file gets the change.
  - For example, a change to how bytes move, or to how the engine verifies the subtree, goes
    to the rsync engine.
  - `Taskfile.yml` and its comments tell the function of the root vars, `FILTER` and the two
    verbs.
  - Do not add shell scripts. To add to a verb of lib, use a hook. Do not make a copy of the
    verb.
  - In `excludes:`, the toolbox include has `report-mirror`, and the engine include has
    `index`. Do not give a new value to a var that lib sets in its `vars:`.
  - Root vars hold only the values of this mirror. Do not put an engine default in a root
    var, because then the command line cannot set it
    ([lib README, Rules a mirror keeps](https://github.com/katoptra/lib#rules-a-mirror-keeps)).
  - Do not put `MAX_BATCHES` or `RECONCILE` in the root vars. In an engine verb, a
    `KEY=value` on the command line cannot change a root var. A run sets them:
    `task sync -- RECONCILE=true`.
- Keep each object of the subtree in `systems/texlive/tlnet/`. The `tlmgr` configuration of
  each user has that path. `SOURCE` is the root of CTAN, and `FILTER` keeps the subtree and
  `/timestamp` in the listing. Thus, each key in the subtree keeps its
  `systems/texlive/tlnet/` prefix. The only other keys of the bucket are in `.state/`, and
  `index.html` and `timestamp` at the root.
- Calculate the storage that a change adds, and the number of uploaded files that it adds.
  Compare them with the 6.8 GB baseline and the 10 GB ceiling. `CEILING_GB: 10` makes
  `split` stop a run if upstream is more than 10 GB.
- GitHub Actions has no cost for a public repository.
- Writing: obey ASD-STE100 and the rules in the
  [Writing section of the org CONTRIBUTING](https://github.com/katoptra/.github/blob/main/CONTRIBUTING.md#writing).
  Read that section before you write.

## Must knows

Each item here prevents an error that occurred, or a cost that can occur. Keep each item.

- **No copy that has a revision number in its name.** `tlmgr` downloads
  `archive/foo.tar.xz`, and upstream keeps `foo.r123.tar.xz` with a symlink to it. R2 has no
  symlinks. Thus, the engine lists and fetches with `-L`, and `FILTER` removes
  `*.r[0-9]*.tar.xz` and the `update-tlmgr-r*` updaters from the listing. Two copies of each
  container make the storage and the cost two times larger.
- **The diff finds a new revision with the same number of bytes.** The state line has the
  mtime of each file, and a new revision has a new mtime.
- **The sequence of `FILTER` is important.** rsync uses the first rule that agrees with a
  path. Thus, the two `--exclude` rules are before the `--include` rules that open the
  subtree, and `--include=/timestamp` is before the last `--exclude`.
- **Freshness is the clock of CTAN.** `FILTER` includes `/timestamp` from the root of CTAN,
  and `FRESH_KEY` is `timestamp`. The subtree does not change for some weeks before each
  TeX Live release. But the master of CTAN writes `timestamp` each hour. If that clock is
  more than 24 hours before the run, `fresh` stops the run with an error. Do not add a
  different freshness check or a notification dependency.
- **The CLI sends each file in one PutObject.** The rsync image of lib sets the multipart
  threshold of the AWS CLI to 4 GiB, in `/etc/aws.config`. The CLI default is 8 MB. The
  largest tlnet file is approximately 145 MB. If the CLI sends a file in parts, it uses three
  or more Class A operations for that file.
- **`index.html` is at the root of the bucket**, not in the subtree. `index` uploads it with
  `Cache-Control: no-cache`. With `OWN: index.html`, `reconcile` does not delete it as a key
  that upstream does not have. A Cloudflare Transform Rule changes `/` to `/index.html`.
  Without that rule, the root gives a 404, but the mirror operates correctly.
- **`site/index.html` is the landing page.** At each run, `index` writes the page with the
  date of the run to `.run/index.html`. It uploads that file only if `sed` had no error.
  Thus, `index` does not upload an empty page. The page gives the same "How to use" as the
  README. Thus, a change to that section of the README is usually also a change to the page.
- **`reconcile` runs in almost all runs.** lib's `due` reads the start time of the last
  reconcile in `.state/reconciled`. It lets `reconcile` run if that time is 23.5 hours or
  more before the run. With one run each day, almost all runs thus make the state again
  from a listing of the bucket. Then `reconcile` deletes each key that upstream, `.state/`
  and `OWN` do not have.
- **In some runs, `reconcile` does not run.** It does not run if the run starts less than
  23.5 hours after the last reconcile. It also does not run in a chained run while batches
  stay in the delta. `task sync -- RECONCILE=true` starts a reconcile manually.
- **Do not count from the job log.** `report` counts from `.run/`, not from the log. To know
  if a run did all its work, do not count files. Use `smoke`.
- **The zone rules are not in this repository.** README step 4 gives the list of them. The
  Configuration Rule sets the three HTML rewriters and Browser Integrity Check to off.
- **The canary finds only Browser Integrity Check.** The canary is
  `systems/texlive/tlnet/tlpkg/texlive.tlpdb.sha512`, and `smoke` reads it as a Perl client
  (`libwww-perl`). If Browser Integrity Check is on again, the run stops with an error. The
  canary is not an HTML file, and the rewriters change only HTML. Thus, the canary does not
  find a rewriter that is on again.
- **The Cache Rule bypasses the edge cache.** `.xz` is not in the default cache list of
  Cloudflare, but `.gz` and `.exe` are in that list. There is no purge step. If a cache rule
  keeps files at the edge, add a purge step before `smoke`.
- **A run that stops with an error is the only alert.** healthchecks.io sends an email if it
  gets no `ping` for one day. Thus, it also finds a scheduler that stops. If upstream is more
  than 10 GB, `split` stops the run before the run fetches a file. Then the mirror does not
  get the changes of that day.

## Verifying a change

All checks run in the toolbox image.

- `task check` makes a render of each command of the pipeline in the image. Then it compares
  the render with `render.txt`. `task render-update` accepts a change. The `check` workflow
  does the same check on each pull request.
- `task run -- task list` lists the subtree through `FILTER`, with no credentials.
  `.run/upstream.txt` must have approximately 17,000 lines: the `systems/texlive/tlnet/`
  paths and `timestamp`. No path agrees with `\.r[0-9]+\.tar\.xz$`, and no file name starts
  with `update-tlmgr-r`. `LIST_FLOOR` is 15000, approximately 90% of the line count.
- `task plan` runs the read-only half on the bucket of the mirror, with `op run`. It uploads
  no file. It does these steps:
  - `list` writes the listing to `.run/`.
  - `state` gets the state from the bucket and writes it to `.run/`.
  - `diff` and `split` make the delta and its batches.
- Do the checks of these verbs in lib, with
  `cd ../lib/examples/rsync && task run -- task offline`:
  - The engine verbs `diff`, `split`, `merge` and `retry`
  - The signature checks `prepare` and `verify`.
- `publish`, `checkpoint`, `delete`, `rebuild` and `index` use credentials. To do a test of
  them on a fork, set `BUCKET` in `Taskfile.yml` to a scratch bucket. Then use
  `task sync -- MAX_BATCHES=1 BATCH_GB=1`. In an engine verb, a `KEY=value` on the command
  line cannot change a root var. Thus, `task sync -- BUCKET=x` does not change the bucket.
- Is it fresh? `curl -s https://tlnet.katoptra.org/timestamp` shows the clock of the master
  of CTAN at the last run (year-month-day-hour-minute, UTC).
