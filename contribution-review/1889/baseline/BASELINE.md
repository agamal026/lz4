# LZ4 #1889: reproduced baseline

The [reported compatibility problem](https://github.com/lz4/lz4/issues/1889)
reproduces on upstream `dev` at
`e7c36223415219c0a18ea75e3fe2eb5d267ee667`.
The [existing legacy-format description](https://github.com/lz4/lz4/blob/e7c36223415219c0a18ea75e3fe2eb5d267ee667/doc/lz4_Frame_format.md)
already describes the Linux kernel's four-zero-byte end marker.

The test-only source is
[`28ca6d54d6e14c8546ebd24eb671b968b568a099`](https://github.com/agamal026/lz4/commit/28ca6d54d6e14c8546ebd24eb671b968b568a099).
Its production files are byte-identical to that upstream base. The shell test
SHA-256 is `36ec36744138a8fcdd462fa3289176c57c6a5fed072528bb45a3daa156701df9`.

Both baseline binaries were built from the unchanged production source with
`make -C programs -j2 lz4 lz4-nomt V=1 CFLAGS='-O2 -Werror'`.
Compiler: GCC 13.3.0, Ubuntu 13.3.0-6ubuntu2~24.04; GNU Make 4.3, Linux x86-64.
The `lz4` target enabled `LZ4IO_MULTITHREAD` and pthreads; `lz4-nomt` omitted
them. The build exited zero. Complete build output was not retained; the
binary hashes and the complete subsequent shell-test logs are retained.

`make -C tests datagen V=1` built the existing fixture generator. For each
mode, its saved baseline executable was exposed as `lz4` first on `PATH`,
followed by the checkout's `tests` directory. From `tests`, run
`sh test-lz4-multiple-legacy.sh`. `LZ4_NBWORKERS=2` bounds the threaded
fixture compressor; the nonthreaded binary reports that threads are unavailable.

On **2026-10-18 at 12:46:52 UTC**, both executions reached the first new
zero-terminated decode and exited **64** with
`Decoding Failed ! Corrupted input detected !`. Existing preceding legacy
controls passed. Because the shell script stops on this error, later new
cases were not executed in these two baseline runs.

The new fixture contains 25 MiB of deterministic data followed by 1 MiB of
sparse data: 27,262,976 uncompressed bytes across four legacy blocks. The
end marker is four zero bytes after the final compressed block.

`baseline-controls.json` separately records file/stdin decoding, integrity
mode, and the listing boundary defect on a small deterministic stream.
The old listing reader accepts a plausible compressed block after the zero
terminator without requiring a new frame magic. That control distinguishes
the listing correction from positive zero-plus-EOF listing, which already works.

Remaining work: recognize zero sizes in both decoders using their existing
cleanup paths, end listing after counting the four-byte marker, run the full
regression with both build variants, and obtain independent automated source
review. No upstream submission, acceptance, human review, or financial benefit
is established by this reproduction checkpoint.
