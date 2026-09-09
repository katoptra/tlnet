# tlnet

[![sync](https://github.com/katoptra/tlnet/actions/workflows/sync.yml/badge.svg)](https://github.com/katoptra/tlnet/actions/workflows/sync.yml)
[![license](https://img.shields.io/github/license/katoptra/tlnet)](https://github.com/katoptra/tlnet/blob/main/LICENSE)
[![mirror](https://healthchecks.io/badge/8955b5d3-ba3b-4e8a-ac39-8501494333f5/otTXcui6-2.svg)](https://github.com/katoptra/tlnet/actions/workflows/sync.yml)

A daily mirror of `CTAN/systems/texlive/tlnet` on Cloudflare R2. This is the directory
`tlmgr` installs and updates from, and it is the only part of CTAN here, complete with every
platform, docs and sources. About 17,000 files and 6.8 GB.

## How to use

TeX Live and TinyTeX both use `tlmgr`:

```sh
tlmgr option repository https://tlnet.katoptra.org/systems/texlive/tlnet/
tlmgr update --self --all
```

For a fresh install, give the installer the same URL:

```sh
install-tl -repository https://tlnet.katoptra.org/systems/texlive/tlnet/
```

To go back to CTAN's mirror rotation: `tlmgr option repository ctan`.

The path mirrors CTAN's own layout, so the host works anywhere a CTAN mirror URL does. It
tracks the current TeX Live release and moves to the next one when upstream does.

## How it works

Once a day GitHub Actions runs the following pipeline inside the toolbox image. Every step
is a verb of the rsync engine in [katoptra/lib](https://github.com/katoptra/lib), in the
order [`Taskfile.yml`](https://github.com/katoptra/tlnet/blob/main/Taskfile.yml) gives
them; the landing page is this mirror's own verb:

1. **`clock` `list` `state` `rebuild`** — stamp the day, list the subtree on CTAN's master
   (dante), and fetch the listing the previous run left in the bucket, rebuilding it from
   the bucket if it went missing.
2. **`diff` `split`** — take what upstream has and the state lacks, refuse a tree past R2's
   free 10 GB, and split the delta into batches.
3. **`prepare` `batches`** — check the signed `texlive.tlpdb.sha512` against the pinned
   TeX Live key, then per batch rsync the files, check every signed file and every package
   container against the tlpdb's checksums, upload, and write the new state. The tlpdb
   lands last, after every container it names.
4. **`delete` `reconcile`** — drop the keys that left upstream, and once a day sweep the
   bucket against the state for anything neither owns.
5. **`index`** — upload the landing page at `https://tlnet.katoptra.org/`, dated.
6. **`smoke` `report` `ping`** — read `texlive.tlpdb.sha512` and a sample of the run's keys
   back over the public domain, summarise what landed, and ping healthchecks.io. Silence
   is the alert.

**Is it fresh?**

Check `last-modified` on the index:

```sh
curl -sI https://tlnet.katoptra.org/systems/texlive/tlnet/tlpkg/texlive.tlpdb.sha512
```

## Why use this?

`tlmgr`'s default repository is CTAN's mirror rotation: every request lands on a different
volunteer mirror, and any one of them can be unreachable, on a slow connection, behind on
TLS, or a few days stale.

This setup ensures a consistent, reliable source for TeX Live updates built on
[Cloudflare's network](https://cloudflare.com/network).

## Want your own?

1. Fork [this repo](https://github.com/katoptra/tlnet).
2. Create an R2 bucket named `tlnet`, an API token with Object Read & Write scoped to it,
   and a custom domain pointing at the bucket. Set `HOST` to that domain in `Taskfile.yml`.
   For a landing page at `/`, add a Cloudflare Transform Rule rewriting the path `/` to
   `/index.html`, and put your own links and text in `site/index.html`.
3. Put the values below in a 1Password vault, point `op.env` at it, and add one repository
   secret, `OP_SERVICE_ACCOUNT_TOKEN`, for a service account that can read the vault. That
   is the whole requirement.
4. Actions -> sync -> Run workflow. The first run finds an empty bucket and fills it; every
   run after that pushes the daily delta.
5. Start it every day. Nothing in this repository schedules a run: add a `schedule:`
   trigger to `sync.yml` with a time of your own, or dispatch it from outside, as this
   mirror is.

The references in `op.env` name one item, `tlnet`, with two sections:

| Section | Field | What it is |
| --- | --- | --- |
| `r2` | `access_key_id` | R2 API token with Object Read & Write on the bucket |
| `r2` | `secret_access_key` | That token's secret |
| `r2` | `endpoint` | `https://<account-id>.r2.cloudflarestorage.com` |
| `healthcheck` | `url` | Optional: a healthchecks.io ping URL |

To test or run locally, with `task`, the 1Password CLI and Docker (or Apple's `container`)
installed:

```sh
task check    # render every command of the pipeline inside the toolbox image; diff it against render.txt
task sync     # one run; op run resolves op.env around it
```

The image, the engine's verbs and the two workflows this repository calls are
[katoptra/lib](https://github.com/katoptra/lib)'s: the include and the image at `v2`, the
workflows at a release commit.

Pull requests are welcome.

MIT licensed. Built by [Josh Vaughen](https://ijosh.com).
