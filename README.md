# WolOS Hub log

The public log of the WolOS Hub: every index that was ever signed, every check verdict, every author certificate the
Authors CA issued, and every revocation. Only published, signed output lives here. Licence GPL-2.0-or-later. Design:
ADR 0047 (the Hub), ADR 0046 (author certificates).

This repository holds no code and runs no workflow. The pipeline that checks releases, signs the index and issues
certificates is kept private (to keep the attack surface small); it writes here with one deploy key per job
environment, and only to the folders below. A commit here is the record of a publication, not a request for one: pull
requests are not taken.

## What is here

```
signed/                          the last publication: index.json, index.json.sig, index.json.wsig, revoked.json, revoked.json.wsig
signed/statements/<id>/<version>.wsig   the signed release statement of each listed version (once the pipeline signs statements)
                                 git history of signed/ = every index ever signed, append-only
verdicts/<id>/<version>.json     one check verdict per release (verdict, findings, sha256, size, run, workflow sha)
certs/<serial>.wolcert           every author certificate the Authors CA issued
certs/<serial>.json              its record: {serial, tier, kind, github_id, login, request_id, issued_at, approved_by+approved_at | cla_accepted_at}
certs/roots/                     the public root certificates (the ones the WolOS image pins)
certs/intermediates/             the intermediate certificates (Index, Authors CA)
revocations.json                 the entries of the revocation list, {serial, at, reason}; signed/revoked.json is made from it
schema/                          the public contract: index.schema.json (I3), revoked.schema.json, request.schema.json
```

Clients read the CDN, not this repository: `https://cdn.wolos.org/plugins/{index.json,index.json.sig,index.json.wsig,revoked.json,revoked.json.wsig}`.
The CDN and `signed/` carry the same bytes; this repository adds the history.

## How to verify

You need `woltrust`, built from the woltrust crate of `wonderlandos/wolos` (`crates/woltrust`,
`cargo build --release -p woltrust --bin woltrust`).

1. Check the roots first. `certs/roots/` must be the roots your WolOS image carries; compare them byte for byte (or their
   fingerprints) with the image's trust store. A root that only this repository vouches for proves nothing.
2. Verify the revocation list and the index:

   ```
   woltrust verify --purpose revocation --wsig signed/revoked.json.wsig \
       --roots certs/roots --intermediates certs/intermediates --list signed/revoked.json signed/revoked.json
   woltrust verify --purpose index --wsig signed/index.json.wsig \
       --roots certs/roots --intermediates certs/intermediates --list signed/revoked.json signed/index.json
   ```

   Each prints a report and exits 0 only when the signature chains to a WolOS root and nothing on the list withdraws it.
   `index.json.sig` is the legacy Ed25519 signature that WolOS 0.12 checks over the same bytes.
3. Check that the CDN serves what was logged:

   ```
   for f in index.json index.json.sig index.json.wsig revoked.json revoked.json.wsig; do
       [ "$(curl -fsS https://cdn.wolos.org/plugins/$f | sha256sum)" = "$(sha256sum < signed/$f)" ] && echo "same: $f" || echo "DIFFERENT: $f"
   done
   ```

4. A release file: `index.json` lists its URL (a GitHub release asset), size and sha256; download it and compare
   `sha256sum`. Its verdict is `verdicts/<id>/<version>.json`. A file signed by an author verifies with
   `woltrust verify` against the same roots and intermediates, the author's certificate from `certs/` as `--chain`,
   and `--list signed/revoked.json`.
5. History: `git log -p -- signed/` shows every index that was signed and when; an index serial never goes down.
   `git log -- revocations.json` shows every revocation; an entry is never edited or removed.

## What revocation means

`key-compromise` and `abuse` void every signature of that certificate, whenever it was made; `superseded` and `ceased`
keep what the index saw before `at`; `withdrawn-version` withdraws one listed version (the serial is the release
statement's) and leaves the author's certificate alone. An online WolOS takes a new list with the next index refresh;
an offline machine keeps the last one it saw (fresh until `expires`, stale up to 30 days after, then verified files
count as unsigned). Revoking an intermediate or a root is done only by a root-signed list, made offline.

## Reporting a problem

A wrong entry, a certificate that should not exist, or a file that is not what its verdict says: write to
hello@wolos.org, or open an issue here. Do not post secrets or private keys in an issue.
