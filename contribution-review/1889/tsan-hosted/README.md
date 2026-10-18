# LZ4 #1889: completed upstream CI after the TSan follow-up

At **2026-10-18 14:11:25–14:11:26 UTC**, all **54 actual jobs, 54 check runs and nine GitHub Actions workflows** for source 0917e70420429ea2acab6aaac48cffdf4e827440 had completed successfully. The AppVeyor pull-request status was also successful. This dated supplement applies to [upstream PR #1890](https://github.com/lz4/lz4/pull/1890).

The complete [upstream TSan job](https://github.com/lz4/lz4/actions/runs/39133845170/job/117694255444) log shows the legacy regression's output comparisons, byte-count and listing assertions, negative controls and final cleanup. The entire instrumented CLI step passed.

[Download the evidence ZIP](lz4-1889-tsan-hosted-evidence.zip). It contains the full decoded TSan log and complete monitor receipt, including actual job and step records, checks, workflows, commit status and canonical source identities. [VALIDATION.json](VALIDATION.json) is the structured audit. [SHA256SUMS](SHA256SUMS) covers the payload; the [archive checksum](lz4-1889-tsan-hosted-evidence.zip.sha256) covers the ZIP.

## Exact source and checkout

| Identity | Value |
|---|---|
| Source head | 0917e70420429ea2acab6aaac48cffdf4e827440 |
| Upstream base | e7c36223415219c0a18ea75e3fe2eb5d267ee667 |
| Source head parent | f83be8241c6042a0f9b500da927f68caa905fc92 |
| Actual CI checkout merge | 9eb241584f5019e048346682cdd8bd2ac76b0100 |
| Source and merge tree | 26cbd5018554777b0b36efd53e76616d9e006b1d |
| Merge parents, in order | e7c36223415219c0a18ea75e3fe2eb5d267ee667, 0917e70420429ea2acab6aaac48cffdf4e827440 |

Every observed job, check and workflow identifies this source head. The TSan checkout prints the merge SHA; canonical Git Data responses establish that the merge and source head have the same tree. The PR head was checked before each observation. Complete responses are in **github-monitor-receipt.json** inside the ZIP.

## Completed workflows

| Workflow | Run ID | Actual jobs | Result |
|---|---|---:|---|
| [Build Systems](https://github.com/lz4/lz4/actions/runs/39133845143) | 39133845143 | 3 | success |
| [CMake Build Tests](https://github.com/lz4/lz4/actions/runs/39133845224) | 39133845224 | 8 | success |
| [Code Quality](https://github.com/lz4/lz4/actions/runs/39133845148) | 39133845148 | 5 | success |
| [Compiler Tests](https://github.com/lz4/lz4/actions/runs/39133845159) | 39133845159 | 7 | success |
| [Core Tests](https://github.com/lz4/lz4/actions/runs/39133845188) | 39133845188 | 9 | success |
| [Cross Platform](https://github.com/lz4/lz4/actions/runs/39133845146) | 39133845146 | 10 | success |
| [OSS-Fuzz](https://github.com/lz4/lz4/actions/runs/39133845150) | 39133845150 | 3 | success |
| [Release & Environment](https://github.com/lz4/lz4/actions/runs/39133845154) | 39133845154 | 4 | success |
| [Sanitizers](https://github.com/lz4/lz4/actions/runs/39133845170) | 39133845170 | 5 | success |

Job IDs and check IDs reconcile exactly. Every completed job step is successful or skipped. The [AppVeyor status](https://ci.appveyor.com/project/YannCollet/lz4-1lndh/builds/54919870) was success, updated at **2026-10-18T14:04:10Z**. AppVeyor is a separate commit-status context and is outside the 54 GitHub Actions jobs.

## Actual TSan regression evidence

Job **117694255444** in run **39133845170** completed successfully at **14:10:01Z**. Its **TSan - CLI test** step ran from **14:00:24Z to 14:09:59Z** and succeeded. The actual commands were:

    make clean
    CC=clang CPPFLAGS="-fsanitize=thread" make -j -C tests test-lz4 V=1

The log shows instrumented compilation and linking, **-DLZ4IO_MULTITHREAD** and **-pthread**, on Ubuntu **24.04.5 LTS**, image **20261012.334.1** and runner **2.337.0**. The compiler command is clang; its exact version is not printed.

The [legacy regression script](https://github.com/lz4/lz4/blob/0917e70420429ea2acab6aaac48cffdf4e827440/tests/test-lz4-multiple-legacy.sh), Git blob **98c4a91c3430d556f7fa3e1151b5960ceaa7e19e**, was invoked at **14:00:34.8831275Z**, reached final true at **14:01:24.2975737Z**, and cleaned up at **14:01:24.3092708Z**.

| Exercised fixture | Decoded bytes | Listed compressed bytes |
|---|---:|---:|
| 26 MiB terminated legacy stream, including sparse tail | 27,262,976 | 12,305,419 |
| Empty terminated legacy stream | 0 | 8 |
| Five-frame concatenation | 27,430,912 | 12,388,233 |

Each fixture has two exact cmp commands, two silent test-mode output assertions, named-file and stdin byte totals, and an exact listing-size assertion. The script also rejects invalid trailing block-like data and its listing, exercises three partial-header cases with expected Error 61, and rejects a corrupt block with expected Error 64. These are intentional negative controls. Assertions and guarded expected failures reach final completion under the script's error handling, and the whole CLI step passes. The full log contains no TSan warning, summary or fatal diagnostic and no make failure.

Concurrent make targets interleave output. The audit uses unique fixture names and exact assertions instead of attributing every nearby stdin count to this script. VALIDATION.json preserves the matched literal lines and timestamps.

## Provenance and limits

This is an **independent automated evidence audit**, not human review. It records upstream execution of the counter correction documented in the [TSan follow-up evidence](https://github.com/agamal026/lz4/blob/a96c638dc9160f43544c8cd8414624cb2c166730/contribution-review/1889/tsan-followup/README.md). Earlier failing runs and baseline evidence remain separate. This success record applies to the exact head, merge tree and observation time above.

The full TSan log was inspected. The other 53 successful jobs are supported by actual job and step records without independently reading every full log. Passing these paths does not establish whole-program race freedom, later status, maintainer approval, merge or reward eligibility. No source edit, rerun, admission action, external comment or remote publication occurred during monitoring or packaging.

The TSan log is **299,883 UTF-8 bytes**, SHA256 **a468dff927a895b094d9a42c0444a600ab49905d8552063ad8144309c218207e**. Its existing BOM and line endings were preserved; it contains no CRLF or replacement characters. Hashes cover the saved encoding of returned decoded text. Original HTTP transport bytes are unavailable. The original monitor receipt is copied unchanged into the ZIP and contains public GitHub material and audit metadata.

## Verify the files

The ZIP contains README.md, VALIDATION.json, tsan-job-117694255444.log, github-monitor-receipt.json and SHA256SUMS. The manifest hashes the first four files. It excludes itself, the ZIP and ZIP checksum file. The same manifest is available beside the ZIP.

    sha256sum -c lz4-1889-tsan-hosted-evidence.zip.sha256
    mkdir evidence
    unzip lz4-1889-tsan-hosted-evidence.zip -d evidence
    cd evidence
    sha256sum -c SHA256SUMS
