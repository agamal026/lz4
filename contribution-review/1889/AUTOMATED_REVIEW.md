# Independent automated review: LZ4 legacy zero terminator

**Disposition:** No material defect or required source change was found in the
reviewed patch for [LZ4 issue #1889](https://github.com/lz4/lz4/issues/1889).
The implementation is technically ready for upstream review, subject to the
separately recorded build and test results and the project's submission process.

This is an **automated source review by a separate agent from the implementation
writer**. It does not establish personal human review. The reviewer did not edit
the implementation, execute tests, audit test logs, or submit the contribution.
Runtime results and upstream acceptance must be assessed from their own evidence.

Review date: **2026-10-18**. The final source identity and all three reviewed file
identities were reconciled at **13:01:49 UTC** on that date.

## Exact source reviewed

| Record | Identity |
| --- | --- |
| [Upstream base](https://github.com/lz4/lz4/commit/e7c36223415219c0a18ea75e3fe2eb5d267ee667) | `e7c36223415219c0a18ea75e3fe2eb5d267ee667` on `dev` |
| [Final contribution source](https://github.com/agamal026/lz4/commit/f83be8241c6042a0f9b500da927f68caa905fc92) | `f83be8241c6042a0f9b500da927f68caa905fc92` |
| Final source tree | `9aaafc0ed65b523f84046e88b09f21e2ddb5b003` |

The review compared the complete three-file diff against the upstream base and
read the surrounding decoder, listing, frame-selection and test code. These
files in the final commit are byte-identical to the files originally reviewed:

| File | Git blob | Mode |
| --- | --- | --- |
| [programs/lz4io.c](https://github.com/agamal026/lz4/blob/f83be8241c6042a0f9b500da927f68caa905fc92/programs/lz4io.c) | `554d3cbc183f0f1c4c11364a95cfbc120010ad29` | `100644` |
| [tests/test-lz4-multiple-legacy.sh](https://github.com/agamal026/lz4/blob/f83be8241c6042a0f9b500da927f68caa905fc92/tests/test-lz4-multiple-legacy.sh) | `98c4a91c3430d556f7fa3e1151b5960ceaa7e19e` | `100755` |
| [doc/lz4_Frame_format.md](https://github.com/agamal026/lz4/blob/f83be8241c6042a0f9b500da927f68caa905fc92/doc/lz4_Frame_format.md) | `c8122eb8ba00e994442b2fd2e681cdd38c25bcc8` | `100644` |

Identity reconciliation used read-only Git operations: `git show -s` for the
commit and tree, and `git ls-tree` for the paths, modes and blobs. No tests were
rerun to prepare this review record.

## Supported problem and implementation scope

The issue requests compatibility with the Linux kernel's four-zero-byte
terminator for legacy LZ4 streams. The upstream base's own
[legacy-frame specification](https://github.com/lz4/lz4/blob/e7c36223415219c0a18ea75e3fe2eb5d267ee667/doc/lz4_Frame_format.md)
already describes that extension. The patch reconciles its EndMark subsection
with the documented extension and explains that a following frame starts with
its own magic number.

Production changes consist of three checks in `programs/lz4io.c`: the threaded
and nonthreaded implementations of `LZ4IO_decodeLegacyStream()`, and the listing
reader `LZ4IO_skipLegacyBlocksData()`. There are no other production changes, new
allocations, public API changes, ABI changes, or changes to compression output.

Before the checks, the decoder source treats a zero block-size word as a
compressed block with zero input bytes. The new checks interpret that word as
the explicit end of the current legacy frame. This source trace supports the
proposed behavior; the actual pre-fix failure and final execution results belong
to the separate validation record.

## Correctness findings

### Decoder boundaries and following frames

Both checks occur after reading a complete four-byte block-size header and
before reading, decoding or submitting any compressed block data. The change
therefore applies to a zero-valued **block header**, not to zero bytes within a
compressed payload.

Breaking at that point leaves the input cursor immediately after the marker.
The existing outer `selectDecoder()` loop can read a subsequent legacy, modern
or skippable frame through its existing dispatch paths. No seeking is introduced
in the decoder; the source supports the same boundary handling for pipes.

The `g_magicRead` mechanism remains appropriate for the older implicit boundary
case, where a next-frame magic has already been consumed as a prospective block
size. A zero terminator belongs to the frame just completed, so it requires no
assignment to that variable.

### Cleanup, queued output and sparse files

Both decoder checks use `break` and retain their existing common exit paths.
The threaded reader still waits for decoding jobs, then waits for the writes
those jobs submit. It subsequently flushes the final sparse extent and frees
the thread pools and allocated buffers. The nonthreaded reader likewise retains
its final sparse flush and buffer cleanup.

No new shared state or ordering change is introduced. In particular, the patch
does not return directly from inside the read loop, which would bypass the
queued-work drain and cleanup.

### Listing boundaries and compressed sizes

The listing check occurs after adding the header and block length to
`totalBlocksSize`. For a zero-sized block this adds exactly four bytes before
breaking. The terminator is therefore included in the preceding frame's
compressed size, and the following frame's magic remains unread for the caller.

This third check is materially useful. The previous listing loop could count a
zero word and continue interpreting subsequent block-looking bytes as part of
the same legacy frame. Treating the word as a real terminator makes the outer
reader validate the next bytes as a new frame header.

An empty terminated legacy frame contains its four-byte magic and four-byte
terminator. Its nonzero block-byte total permits the existing listing caller to
report it as an eight-byte frame without another production change.

## Regression-source assessment

The added assertions in the existing shell test cover the significant branches
of this change. This section describes what the **test source exercises**, not
a claim that the reviewer executed it.

| Test case or assertion | Purpose |
| --- | --- |
| 25 MiB generated data plus 1 MiB from `datagen -P100` | Crosses several eight-MiB legacy blocks and includes sparse data before termination. |
| File decoding followed by `cmp` | Checks complete output contents and length. |
| Piped decoding followed by `cmp` | Exercises the nonseekable input route. |
| File and piped `-t`, with empty stdout required | Checks integrity-only operation without decoded output. |
| Magic plus zero word, eight bytes total | Exercises a terminator before the first data block. |
| Empty-frame verbose listing equals `1 8` | Checks the frame count and exact compressed size. |
| Mixed legacy, empty terminated legacy, skippable and modern frames | Checks that decoding continues across supported frame boundaries and excludes skippable payloads from output. |
| Verbose frame-type sequence | Checks that listing keeps the concatenated frames distinct. |
| Sum of verbose compressed sizes equals `wc -c` | Checks that headers, including the terminator, are counted. |
| Terminator followed by block-looking bytes without a new magic | Requires both decoding and listing to reject continuation as part of the old frame. |
| One-, two- and three-byte incomplete headers | Preserves rejection of partial size headers. |
| A complete size header followed by invalid compressed bytes | Preserves a genuine compressed-block corruption failure. |

The original portion of the test remains in place and covers ordinary legacy
files and concatenation without an explicit terminator. The existing Makefile
target already runs this shell test, so no new test registration is needed.

The independent review identified an important baseline distinction: listing a
valid frame followed by zero and EOF already happened to succeed before this
patch. Listing zero followed by a valid next-frame magic also already happened
to work. The invalid-tail control specifically distinguishes the listing fix
from those previously successful cases. Its actual baseline exit status should
be taken from the separate recorded execution evidence.

Both threaded and nonthreaded builds need actual validation because their
production decoding loops are separate implementations. The multiblock fixture
is relevant to the threaded exit path; its complete-output comparison checks
that queued writes and final sparse output are preserved.

## Shell portability assessment

The additions use POSIX shell functions, loops, redirection and command
substitution, together with standard `printf`, `awk`, `wc`, `cmp` and `test`.
Binary framing is expressed through octal `printf` escapes. The suffixes used
as `printf` format strings are fixed literals in the script, not externally
supplied text.

The test follows the existing temporary-file prefix and cleanup convention.
Its aggregate sizes remain comfortably within signed 32-bit integer range for
the shell and listing assertions. No additional interpreter or dependency is
introduced. This is a source portability assessment; successful execution on
particular shells or operating systems requires its own validation evidence.

## Limits and remaining evidence

- This is an automated review, with no claim of personal human review,
  maintainer approval, merge, payment, or benefit eligibility.
- The reviewer did not execute tests or audit build logs. Build success,
  regression exit statuses, sanitizer results and platform coverage must be
  read from the implementation writer's separately recorded evidence.
- The review covers the three identified files and the surrounding paths needed
  to assess them. It is not a comprehensive audit of LZ4 or its thread pool.
- Accepting a zero-size word intentionally expands the set of valid legacy
  streams. The negative controls support specific retained failure behavior;
  they do not establish that every previously rejected byte stream remains
  rejected.
- `--list` scans frame structure; it does not validate compressed payloads. Its
  preexisting ability to seek beyond some truncated block payloads is outside
  this change.
- A legacy magic followed immediately by EOF has a preexisting difference
  between empty decoding and listing. This patch's explicit empty terminated
  frame case does not require changing that separate behavior.
- The issue's linked downstream mkinitcpio discussion was not independently
  retrieved by the reviewer. The upstream LZ4 specification itself supports
  the kernel extension used here.
- No patch-specific security or financial reward qualification is established
  by this compatibility fix or this review.

**Final assessment:** The reviewed checks have the correct scope and placement,
preserve cleanup and following-frame handling, and are backed by meaningful
regression assertions. No additional source change or regression was required
by this independent automated review. Submission readiness remains conditional
on the actual validation results for the exact source identity above.
