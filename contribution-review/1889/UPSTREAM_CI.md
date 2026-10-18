# LZ4 #1890: bounded upstream CI observation

At **2026-10-18 13:13:17–13:13:26 UTC**, [upstream PR #1890](https://github.com/lz4/lz4/pull/1890) remained open, non-draft and unmerged at source `f83be8241c6042a0f9b500da927f68caa905fc92`, based on `dev` at `e7c36223415219c0a18ea75e3fe2eb5d267ee667`. Maintainer branch modification remained disabled and PR metadata reported zero comments and zero review comments.

| Observed collection | Actual result |
| --- | --- |
| 54 CheckRuns | 48 successful; 6 in progress; no failures |
| All 54 actual workflow jobs and their step metadata | 48 successful; 6 in progress |
| Nine upstream workflow runs | Six successful; three in progress |
| [AppVeyor build 54919754](https://ci.appveyor.com/project/YannCollet/lz4-1lndh/builds/54919754) through GitHub commit status | Pending; provider logs not inspected |

The unfinished jobs were Cppcheck Static Analysis, Valgrind Memory Analysis, macOS, AddressSanitizer (x64), ThreadSanitizer (x64), and MemorySanitizer (x64). They belong to Code Quality, Cross Platform and Sanitizers. The complete matrix was **still pending**. No failed job logs were requested because no failure appeared; no failure was attributed to the baseline blocksize comparison.

The [compressed observation](upstream-ci-2026-10-18.zip) contains the complete selected canonical API observations, all job/step summaries, source URLs, collection coverage, pending names and limitations. The two archive members are `observation.json` and `SHA256SUMS`; these are archive-internal paths, not separate repository files.

- Archive: 13270 bytes; SHA-256 `7d07820fa991a6735f038b6412b79f37927509e63a0b59753770e9c121b3934f`.
- `observation.json`: 225056 bytes; SHA-256 `02eee9917be10d2e3d57a1d02b329ba0b47be6690c9cc67e37217a5ea63632d2`.

The endpoints were collected over the stated window, not atomically. All collection pages used `per_page=100` and their returned counts matched reported totals. This automated observation reads actual API job and step metadata. It does not establish execution of every individual regression case from successful job logs, which were not retrieved. It is not human review, maintainer acceptance or realized financial benefit.

The [immutable local validation packet](https://github.com/agamal026/lz4/blob/9016232b3f5ef31764c6b4ae30f201da1ecaa445/contribution-review/1889/README.md) remains separate. It records passing threaded/nonthreaded focused tests and the ten-test listing suite, plus the broader CLI blocksize failure reproduced with unchanged upstream source. Do not relabel that local broader command as passing based on this hosted observation. No source change, workflow rerun, upstream comment or fork Actions activation was performed for this follow-through.
