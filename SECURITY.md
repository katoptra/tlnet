# Security

## What this mirror guarantees

Every daily run verifies what it publishes, batch by batch, before publishing it:

- `texlive.tlpdb`, the three installers and the two `update-tlmgr-latest` updaters are
  checked against their SHA-512 and GPG signatures, with the TeX Live primary key
  fingerprint pinned in `Taskfile.yml`. A signature from an expired or revoked key is
  rejected.
- `texlive.tlpdb.xz`, the copy `tlmgr` downloads, must decompress to the verified
  `texlive.tlpdb` byte for byte.
- Every package container is checked against the checksum recorded in the signed tlpdb, and
  the tlpdb goes live only after every container it names is in the bucket.
- A batch that fails any check is not uploaded; the bucket stays at its last checkpoint.
- After each publish, `texlive.tlpdb.sha512` and a sample of the run's keys are read back
  through the domain and compared with what was uploaded.

Those are the only files TeX Live signs. The rest of the tree (`install-tl`,
`install-tl-windows.bat`, `texlive.tlpdb.md5` and the helper trees under `tlpkg/`) is
copied as served; no TeX Live tool fetches them from a repository URL. `tlmgr` repeats the
signature check on the client, so a tampered mirror is rejected there too.

Uploads are batched. Containers land before the tlpdb that names them, so a `tlmgr` run
that overlaps a publish sees the previous tlpdb, never a tlpdb naming a file that is not
there.

## Reporting

If you find a way to serve altered or unsigned content through `tlnet.katoptra.org`, or a
weakness in the pipeline itself, report it privately through
[GitHub's vulnerability reporting](https://github.com/katoptra/tlnet/security/advisories/new).
Please do not open a public issue for it. Problems with the packages themselves belong to
[TeX Live](https://tug.org/texlive/) and [CTAN](https://ctan.org/); this mirror copies what
they publish.
