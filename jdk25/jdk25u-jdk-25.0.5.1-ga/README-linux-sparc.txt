Linux/sparc64 patch lists

The .pls.sparc and .pls.linux lists add a JIT port for Linux on sparc64
on top of the SunOS lists. Apply them in order: the version's .pls, then
.pls.sparc, then .pls.linux. The sparc list is OS-neutral cpu/sparc code
and is meant to be useful for Solaris SPARC as well; the linux list holds
os_cpu/linux_sparc.

Tested on a SPARC T7-1 (M7, 256 strands) running Gentoo with Linux 7.3 and
GCC 16.2, with jtreg tier1 on both release and fastdebug builds.

Building

    bash configure --with-toolchain-type=gcc --disable-warnings-as-errors \
        --enable-unlimited-crypto --with-boot-jdk=<jdk25>

A release build takes about 7.5 minutes with make -j96 on the T7.

What works

- C1 and C2, interpreter, CDS. Serial, Parallel and G1 (the default). No
  ZGC or Shenandoah.
- Lightweight locking is the only locking mode (UseObjectMonitorTable is
  off).
- Virtual threads run on real continuations, including monitor preemption
  (JEP 491).
- A native FFM linker for the Linux sparc64 ABI. The libffi fallback
  linker can still be configured but is not used.
- Intrinsics: AES (ECB, CBC, CTR), GHASH, SHA-1/256/512, CRC32, CRC32C,
  Adler32, multiplyToLen, Montgomery multiply/square, and the string
  intrinsics CountPositives, StrIndexOf, StrIndexOfChar, EncodeISOArray,
  compress/inflate and the array/string equals. VIS3 (T4 and later) is used
  where it helps.
- UDivI/UModI/UDivL/UModL and the spin-wait hint (pause).

Not supported

- The serviceability agent (SA). Its tier1 tests fail.
- Compact object headers. -XX:+UseCompactObjectHeaders warns and is turned
  off.
- JVMCI/Graal.

Known test failures that are not defects

- Unsafe tests that assume unaligned access (SPARC is strict-alignment).
- Tests that assume user addresses below 2^63. sparc64 Linux maps the
  stack and some mappings above it.
- Code cache sizing tests on a host this large, LM_MONITOR/JNIMonitor
  variants, IR shape tests that expect x86/aarch64 addressing or wider
  vectors, and TestNoNULL hits in the Solaris files.
- A few tests time out under load (LargeValueExceptions,
  TestAlignVectorFuzzer, Continuation Fuzz).

Notes for applications

- GraalJS through javax.script fails at engine creation because there is
  no libtruffleattach for linux-sparcv9. Start the JVM with
  -Dtruffle.attach.library=/dev/null
  -Dpolyglotimpl.AttachLibraryFailureAction=warn
- Libraries with native code still need sparc64 .so files.
- The JVM is container-aware, so in Docker the CPU limit sizes the GC,
  compiler and ForkJoin thread pools.
