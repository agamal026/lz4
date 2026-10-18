# LZ4 #1890: legacy progress counter race correction

Source: [`0917e70420429ea2acab6aaac48cffdf4e827440`](https://github.com/agamal026/lz4/commit/0917e70420429ea2acab6aaac48cffdf4e827440), tree `26cbd5018554777b0b36efd53e76616d9e006b1d`, parent `f83be8241c6042a0f9b500da927f68caa905fc92`.

The admitted [upstream TSan job](https://github.com/lz4/lz4/actions/runs/39130134533/job/117683433807) found a write/read race between the legacy decoder's byte-count update and the separate writer's progress display. All 54 jobs on the previous source completed: 53 succeeded and this one failed; AppVeyor's GitHub status reported success. The full failed-job log is retained in the archive without modification.

The coordinator [reproduced the same race on unchanged upstream](https://github.com/agamal026/lz4/blob/a7f764a75f381654296c2d6e55d9c89478c21400/contribution-review/1889/tsan-baseline/README.md) using an ordinary 26 MiB legacy stream with no end marker. That control exited 66 with one TSan warning. The end-marker regression exposed an existing ownership problem.

## Correction

The private write-job counter pointer is mutable, and the counter increment moves from the decoder to the existing sole writer, immediately before its progress read. This follows the modern frame writer's existing ownership pattern. Both pool completion waits remain in place before the final count is returned. The change is two additions and two deletions in one source file. No test, sanitizer suppression, locking primitive, public API, allocation, compression or listing change is added.

## Actual local validation

Linux x86-64, GCC 13.3.0; executed on October 18, 2026, 13:53:09–13:53:50 UTC.

| Check | Result |
| --- | --- |
| ThreadSanitizer build, `CPPFLAGS=-fsanitize=thread`, `CFLAGS=-O1 -g -Werror` | Exit 0 |
| Same ordinary 26 MiB decode that failed on unchanged upstream | Exit 0, 27,262,976 decoded bytes, no TSan warning |
| Byte-for-byte comparison of decoded fixture | Exit 0 |
| Complete unchanged `test-lz4-multiple-legacy.sh` under ThreadSanitizer | Exit 0, no TSan warning |
| Threaded and nonthreaded builds with `CFLAGS=-O2 -Werror` | Exit 0 |
| Same complete regression with the normal threaded executable | Exit 0 |
| Same complete regression with the normal nonthreaded executable | Exit 0 |
| `git diff --check` | Exit 0 |

The regression covers ordinary and terminated legacy frames, a 26 MiB multi-block sparse-tail fixture, empty frames, file and piped decoding, integrity mode, concatenation, listing totals and malformed-input controls. The nonthreaded executable's hash is unchanged from the earlier validated source. Negative controls intentionally print decode errors while the complete script succeeds. The reviewed change specifically moves byte accounting; logs retain all decoded totals and progress output.

The [independent automated review](AUTOMATED_REVIEW.md) checked the exact changed-file hash, both relevant thread-pool completion implementations, test/empty/sparse semantics, and the established modern-writer pattern. No required correction was found. That reviewer did not execute tests or audit runtime logs; execution is separately recorded above.

## Files and limits

[VALIDATION.json](VALIDATION.json) records exact commands, environment overrides, times, exits, source/binary/log hashes and normalization. [source.patch](source.patch) is the incremental source change. [validation-logs.zip](validation-logs.zip) contains the full local logs, previous upstream observation and failed-job log, plus the direct payload files. Paths named as archive members are inside this ZIP, not separate GitHub blob paths. [SHA256SUMS](SHA256SUMS) lists every archive payload file except itself; [ARCHIVE_SHA256.txt](ARCHIVE_SHA256.txt) authenticates the archive separately.

Local logs replace only absolute source/evidence/workspace prefixes; original and published hashes are recorded. Fixtures, executables and private workspace metadata are excluded. This packet establishes no new hosted pass for the corrected source: subsequent PR publication and CI are separate observations. The prior broader CLI blocksize failure remains explicitly recorded in [the original packet](https://github.com/agamal026/lz4/blob/9016232b3f5ef31764c6b4ae30f201da1ecaa445/contribution-review/1889/README.md); it was not rerun or relabeled here. No whole-program race-freedom claim, human review, acceptance or reward eligibility is established.
