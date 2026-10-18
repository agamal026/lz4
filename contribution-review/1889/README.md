# LZ4 #1889: legacy zero end markers

This packet records an independently reviewed compatibility fix and its actual
local validation for [LZ4 #1889](https://github.com/lz4/lz4/issues/1889).

## Exact source and reproduction

| Item | Immutable revision |
| --- | --- |
| Upstream `dev` base | [`e7c36223415219c0a18ea75e3fe2eb5d267ee667`](https://github.com/lz4/lz4/commit/e7c36223415219c0a18ea75e3fe2eb5d267ee667) |
| Test-only failing baseline | [`28ca6d54d6e14c8546ebd24eb671b968b568a099`](https://github.com/agamal026/lz4/commit/28ca6d54d6e14c8546ebd24eb671b968b568a099) |
| Final source | [`f83be8241c6042a0f9b500da927f68caa905fc92`](https://github.com/agamal026/lz4/commit/f83be8241c6042a0f9b500da927f68caa905fc92) |
| Final source tree | `9aaafc0ed65b523f84046e88b09f21e2ddb5b003` |
| Baseline execution evidence | [Baseline packet](https://github.com/agamal026/lz4/blob/f9f961ec521cb16ef602e0178f0f19a9f899f058/contribution-review/1889/baseline/BASELINE.md) |

The final shell test is byte-identical to the test-only baseline. Baseline
production code is byte-identical to upstream. Both baseline build variants
exited **64** at the first new zero-terminated decode. Separate baseline
controls also reproduced file/stdin/integrity failures and showed the old
listing reader consuming block-like data after a terminator.

The final change adds one zero-size check to each legacy decoder and one to
the legacy listing reader. Existing decoder cleanup drains queued work and
flushes sparse output. Listing counts the four-byte marker before stopping;
the next frame must start with its own magic number. The format document is
updated to describe the existing Linux kernel extension consistently.

## Executed validation

Linux x86-64; GCC 13.3.0 (Ubuntu 13.3.0-6ubuntu2~24.04); GNU Make 4.3.
All times below are UTC on 2026-10-18. Full receipts give microsecond timestamps.

| Check | Actual result | Time |
| --- | --- | --- |
| `make -C programs --no-print-directory -j2 lz4 lz4-nomt V=1 'CFLAGS=-O2 -Werror'` | Exit 0 | 12:52:02–12:52:04 |
| Extended legacy shell regression, threaded binary | Exit 0 | 12:52:04–12:52:05 |
| Same regression, nonthreaded binary | Exit 0 | 12:52:05–12:52:06 |
| Existing `python3 tests/test-lz4-list.py` suite | 10 tests; zero failures/errors/skips; exit 0 | 12:54:07 |
| Broader CLI essentials/sparse/skippable command | Exit 2 at `test-lz4-blocksize` | 12:53:56–12:54:07 |
| Unchanged upstream binary running the same blocksize test | Exit 1, same byte-5 difference | 12:55:28–12:55:33 |

The regression covers a 26 MiB stream (four legacy blocks, including a sparse
tail), an empty terminated frame, and five mixed concatenated frame types.
Each fixture exercises file and piped decoding, file and piped integrity
testing, and listing size accounting. It checks exact empty-frame size and
frame order. Negative controls reject block-like trailing bytes, partial
one/two/three-byte headers, and an actually corrupt compressed block.
The existing legacy controls remain in the script.

Build the existing `datagen` target with `make -C tests datagen V=1`. For each
mode, expose its saved executable as `lz4` first on `PATH`, followed by the
checkout's `tests` directory. From `tests`, run
`sh test-lz4-multiple-legacy.sh`. Set `LZ4_NBWORKERS=2` for the threaded run;
leave it unset for the final nonthreaded run. Receipt command labels are
checkout-relative; the scripts actually ran from the `tests` directory.

The broader command was:

```sh
make --no-print-directory -C tests -j2 test-lz4-essentials test-lz4-sparse test-lz4-skippable 'CFLAGS=-O2 -Werror'
```

It failed comparing high-compression linked-block output from `-T1` and `-T4`:
`tmp-tbs-T1.lz4 tmp-tbs-T4.lz4 differ: byte 5, line 1`. The saved, unchanged
upstream threaded binary reproduced the same failure with the unchanged
`test-lz4-blocksize.sh` script and the same fixture generator. This is evidence
of a pre-existing broader-suite failure; **the broader suite did not pass**.
No compression code changes are in this patch. The isolated listing suite
used a dedicated temporary directory.

## Review, integrity, and limits

[AUTOMATED_REVIEW.md](AUTOMATED_REVIEW.md) records independent automated source
review of the exact three file blobs, with no required changes. The reviewer
did not execute tests; the implementation agent ran the recorded commands.
This is not a claim of personal or maintainer review.

[VALIDATION.json](VALIDATION.json) records commands, results, source and binary
hashes, timestamps, and log hashes. [SHA256SUMS](SHA256SUMS) covers all archive payload
files except itself. `source.patch` is the exact upstream-to-final diff.
`validation-logs.zip` contains the same allowlisted public packet files;
`ARCHIVE_SHA256.txt` identifies the archive. Binaries and generated fixtures
are excluded.

Only absolute checkout/evidence directory prefixes in `cli-essentials.log`
and `listing-suite.log` were replaced with `<checkout>` and `<evidence>`.
All remaining output bytes were retained. Original command receipts keep
original log hashes; `VALIDATION.json` maps each original hash to the hash
of its published copy. Complete baseline build output was not retained,
as disclosed in the immutable baseline packet.

This evidence establishes the local focused build/regression/listing results.
It does not establish hosted CI, cross-platform or sanitizer results, a full
upstream suite pass, upstream acceptance, reward eligibility, or payment.
