# MRPack Modloader Class Indexer

Java 17+, Maven. Build: `mvn package`.

Run:

```sh
java -jar target/mrpack-indexer-1.0.jar --pack sodium-plus --threads 16 --db /absolute/path/modusage.sqlite --html report.html
java -jar target/mrpack-indexer-1.0.jar --pack /absolute/path/pack.mrpack --db /absolute/path/modusage.sqlite --html report2.html
java -jar target/mrpack-indexer-1.0.jar --view /absolute/path/modusage.sqlite
```

`--pack` accepts a local MRPACK, direct HTTPS MRPACK URL, or Modrinth project slug/ID. For slugs, it selects the first MRPACK in the API version list. The cache persists downloads. SQLite appends a separate run every time. GUI aggregates references across runs (repeat scans are counted as separate runs). Class counts are distinct mods per run, not number of bytecode references. References include bytecode method/field owners, descriptors, superclasses, interfaces, annotations and selected constant pool instructions. Reflective strings and deeply nested bundled jars are not indexed.

## DAX caveat
Oracle SPARC DAX is **not** automatically used by Java's ZIP/Inflater or SHA-512 routines. This implementation is multithreaded and portable, but has **no hardware DAX integration**. A genuine DAX backend requires a supported Solaris native interface and verification of which operations the particular DAX generation accelerates. Do not claim DAX acceleration from running on SPARC alone.

## Security/limitations
SHA-512 verification is used for manifest downloads where available. URLs are restricted to a small set of hosts; redirects are handled by Java HTTP client. Use trusted MRPACK files, since zip bomb limits are not comprehensive. The pack download itself is not checksum-verified for arbitrary URLs. External databases are not supported; SQLite file paths only. Not yet tested on Solaris SPARC.

## Oracle DAX Stream Offload mode

The optional `--dax true` flag invokes `com.oracle.stream.DaxIntStream.of(...).parallel().filter(...).count()` for class-reference aggregation. This is a **real call into the Oracle API** when the original Oracle runtime is supplied; it is not a hardware guarantee for every operation. The compile-time dependency is the `comoraclestream-dax-stub` project, scoped `provided`, so it is **not bundled** into the shaded JAR.

On SPARC Solaris install `library/java/streamoffload`, and launch with the original Oracle JAR first on the classpath:

```sh
java -Djava.library.path=/usr/lib/streamoffload \\
  -cp '/usr/lib/streamoffload/comOracleStream.jar:target/mrpack-indexer-1.0.jar' \\
  org.example.Indexer --pack example.mrpack --threads 32 --dax true \\
  --db /absolute/path/mods.sqlite --html report.html
```

**Important:** The GitHub stub only delegates to standard Java streams. Do not place the stub on the runtime classpath when expecting DAX. Oracle Stream Offload decides whether a supported stream pipeline is actually accelerated. Class parsing, network downloads, and ZIP decompression remain CPU operations. This DAX aggregation is intentionally straightforward rather than optimized for speed; it scans all references once per class and can be slower than the CPU hash-map method. Verify hardware usage with Solaris platform monitoring.

For portability, omit `--dax true` to use the original CPU aggregation.
