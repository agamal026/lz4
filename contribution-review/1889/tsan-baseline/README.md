# Legacy progress counter race: unchanged-upstream reproduction

Observed on October 18, 2026, 13:41:37–13:41:44 UTC. This packet records a reproduction before any race fix.

Upstream PR [#1890](https://github.com/lz4/lz4/pull/1890) exposed a data race in [ThreadSanitizer job 117683433807](https://github.com/lz4/lz4/actions/runs/39130134533/job/117683433807). Its decoder increments `streamSize` while the separate writer reads it for progress output.

A clean checkout of unchanged upstream `e7c36223415219c0a18ea75e3fe2eb5d267ee667` reproduced the same race using GCC 13.3.0 on Linux x86-64, before implementing a correction. The ordinary 26 MiB legacy fixture has **no zero end marker**, so this control exercises the pre-existing decoder independently of the new marker support.

Build from that base checkout:

```sh
CPPFLAGS='-fsanitize=thread' make -C programs --no-print-directory -j2 lz4 CC=gcc V=1 'CFLAGS=-O1 -g -Werror'
```

Generate 25 MiB with `datagen -s4 -g25M`, append 1 MiB from `datagen -g1M -P100`, and use the normal executable at submitted source `f83be8241c6042a0f9b500da927f68caa905fc92` to compress it with `-l -f`. Decode it using the instrumented unchanged-upstream executable with `-d -f`. Build and compression returned 0; decoding returned **66**, with one ThreadSanitizer warning naming `LZ4IO_decompressBlockLegacy` and `LZ4IO_writeDecodedChunk`.

[BASELINE.json](BASELINE.json) retains exact command receipts, source identities, binary and fixture hashes, times, statuses, and original/published log hashes. [baseline-tsan-decode.log](baseline-tsan-decode.log) contains the full observed warning. Published logs normalize only absolute workspace prefixes. Build output is retained; fixtures and executables are excluded. This packet does not claim a corrected source, passing sanitizer run, complete matrix, human review, acceptance or reward eligibility.
