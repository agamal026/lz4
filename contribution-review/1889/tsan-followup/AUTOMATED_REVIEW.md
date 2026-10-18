# Independent automated review — LZ4 legacy byte-counter race correction

**Result:** No required source correction found in the reviewed incremental change. This is an independent AI-assisted source review, not a human review or an independent test run.

## Exact reviewed source

| Record | Identity |
| --- | --- |
| Review observation window | 2026-10-18, 13:51:43–13:53:07 UTC |
| Repository | `lz4/lz4`, contributor checkout |
| Comparison commit | `f83be8241c6042a0f9b500da927f68caa905fc92` |
| Original issue-work upstream base | `e7c36223415219c0a18ea75e3fe2eb5d267ee667` |
| Changed file | `programs/lz4io.c` |
| Reviewed working-file SHA-256 | `47f8169a9883ab1155d6783d4c09979b4b2edd822299f6c5cbaa376bf35c7f42` |
| Reviewed working-file Git blob | `64953a23bf10a09259b5d854d1c1c01a2cd9c69d` |
| Same file at comparison commit | Git blob `554d3cbc183f0f1c4c11364a95cfbc120010ad29` |
| Incremental diff | One file, two additions and two deletions |

The file hashes were checked twice during the review and agreed. The source was an uncommitted working-file change when reviewed; this record does not invent a later source commit. The coordinator must associate these exact bytes with the eventual published source revision.

The diff changes the private `ChunkToWrite.totalSize` field from a pointer to const data to a mutable pointer, removes the counter increment from `LZ4IO_decompressBlockLegacy`, and adds the same accounting to `LZ4IO_writeDecodedChunk` after sparse writing and before the progress display. No other tracked source file was modified at the observed checks.

## Concurrency and lifetime reasoning

### The writer owns the shared counter

Before this increment, the legacy decoder worker incremented `streamSize` through `LegacyBlockInput.totalSize`, while the distinct writer worker read that same object for progress reporting. Having one decoder worker did not serialize the decoder with the writer.

After the change, the decoder only copies the pointer and the nonnegative decoded length into the separately allocated write-job descriptor. It no longer dereferences the counter. The existing writer pool is constructed with `TPool_create(1, 1)`, so both the increment and its immediately following progress read execute on the same worker. The change requires neither an atomic counter nor an additional lock for that access pattern.

The descriptor's `size` is set from `decodedSize` only after the decoder rejects a negative result. The writer adds the same decoded length previously added by the decoder, once per submitted write job. Moving that addition does not change the unsigned accumulator width or its pre-existing overflow behavior.

### Completion precedes the caller's read

`LZ4IO_decodeLegacyStream` initializes `streamSize` and `storedSkips` before any submission. The main reader passes their addresses but does not read or modify the counter while work is outstanding.

On every normal stream-end path, it first calls `TPool_jobsCompleted(tPool)`, then `TPool_jobsCompleted(wPool)`. The first wait ensures that all decoder callbacks have returned and all their write jobs have been submitted. The second wait therefore drains a closed set of write jobs. The caller flushes the final sparse tail and frees the pools and buffers before returning `streamSize`. Neither the counter nor the sparse state can leave scope while a writer still uses it under these existing completion rules.

The pthread implementation waits for both an empty queue and zero busy workers while holding the queue mutex. A callback returns before its worker decrements the busy count under that mutex. The Windows implementation decrements its interlocked pending-job count only after the callback returns; its completion wait rechecks that count rather than relying on the event alone. Its zero-job path does not wait for an event that was never signaled. These source-level completion paths support the required ordering on both implementations. Neither implementation was executed by this reviewer.

The single-threaded fallback pool invokes callbacks synchronously. The nonthreaded legacy decoder itself is a separate preprocessor branch and was not changed by this diff.

### Match with the modern frame writer

The existing `LZ4IO_writeDecodedLZ4FChunk` already performs sparse output, adds the output buffer's size to its total, and then displays progress on its sole writer worker. The new legacy ordering matches that established ownership pattern. The modern decoder and its buffer-pool behavior are unchanged.

## Byte-count and stream-boundary cases

| Case | Source-level assessment |
| --- | --- |
| Ordinary legacy stream ending at EOF | Every valid decoded block still creates one write job. Both pools complete before the sum is returned. |
| Kernel zero-size terminator after data | The reader's existing terminator break still flows through both waits, so queued decoded data remains counted. |
| Empty legacy frame | No block job is created. The initialized counter remains zero; both completion implementations handle zero pending jobs. |
| Valid block that decodes to zero bytes | Its job adds zero, preserving the previous arithmetic. |
| Sparse output and an all-zero tail | The writer adds the full decoded chunk length, including bytes represented as holes. Deferred zero bytes are therefore already counted once; the final sparse flush establishes the file extent without adding them again. |
| Integrity/test mode | `lz4cli.c` sets test mode, selects `nulmark` as output, then uses decompression. The legacy path still executes its writer against the opened null device, so the total remains the decoded byte count. This change does not introduce a test-mode bypass or write decoded bytes to standard output. |
| File and piped input | Input selection does not alter writer ownership or the completion sequence. |
| Concatenated legacy/modern/skippable frames | `selectDecoder` completes each decoder before the outer loop adds its returned byte count and starts the next frame. Per-stream counters cannot be accessed by a prior writer after return. |
| Corrupt block or write failure | Existing error paths terminate the process. The increment now occurs only after `LZ4IO_fwriteSparse` returns successfully; no successful final count is promised on a terminating error path. |
| Nonthreaded legacy decoding | Its counter, writes, and cleanup are unchanged. |

Progress now reflects chunks processed by the output writer instead of permitting the decoder's count to run ahead. For sparse output, that remains a logical decoded-byte count, not a measure of allocated disk blocks or durable storage writes.

## Scope and limits

The review covered the exact incremental diff, all accesses to the legacy stream-size counter, write-job construction and lifetime, sparse output and tail handling, the modern frame-writer comparison, decoder selection and aggregation, test-mode destination selection, and the pthread, Windows, and synchronous thread-pool completion implementations.

The relevant unchanged comparison-commit blobs were:

| File | Git blob |
| --- | --- |
| `programs/threadpool.c` | `60200d3b88a4a004526a05d464e45b63e9a83bbd` |
| `programs/threadpool.h` | `940754a7b237f08bc0acff6b5b2cfd655ef130c3` |
| `programs/lz4cli.c` | `460372c4340d420259d166d263805a4ae7c97379` |
| `tests/test-lz4-multiple-legacy.sh` | `98c4a91c3430d556f7fa3e1151b5960ceaa7e19e` |

No test, compiler, sanitizer, build, fixture-generation, publication, or external messaging action was performed by this reviewer. The regression script's unchanged identity was checked; the script was not executed or independently re-audited in this packet.

The coordinator reported a GCC 13 ThreadSanitizer reproduction on unchanged upstream `e7c36223415219c0a18ea75e3fe2eb5d267ee667` using an ordinary unterminated 26 MiB legacy input, with baseline evidence commit `a7f764a75f381654296c2d6e55d9c89478c21400`. This reviewer did not audit that evidence packet or its logs and does not independently certify its runtime result. The same-control rerun, focused threaded/nonthreaded regressions, and ThreadSanitizer results on the correction remain the coordinator's separate validation responsibility.

No remaining counter race introduced by this increment was found in the inspected source. This is not a whole-program race-freedom claim, a proof of platform runtime results, a disposition of unrelated CI failures, human approval, upstream acceptance, or reward eligibility.
