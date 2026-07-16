# d8.jar — built from xiaoyvyv/android-d8

A working, self-contained `d8.jar` (class → dex compiler), built from
[xiaoyvyv/android-d8](https://github.com/xiaoyvyv/android-d8), which mirrors
Google's real D8/R8 dexer source (`com.android.tools.r8.*`, matching the
upstream project's own package layout and license headers).

## How it was built

1. Cloned the repo — 592 `.java` files under `src/`, plus prebuilt
   dependency jars (`asm-all`, `guava`, `commons-compress`, `fastutil`,
   `jopt-simple`, `org.json.simple`) under `libs/`.
2. Compiled the full source tree with `ecj` (Eclipse Compiler for Java)
   against those dependency jars as classpath. Clean compile, zero errors.
3. Merged the compiled classes with all dependency classes into one
   directory (uber-jar layout), added a manifest with
   `Main-Class: com.android.tools.r8.D8`, and zipped it up.

## One patch applied

`com/android/tools/r8/utils/ThreadUtils.java` sized its thread pool as
`min(availableProcessors(), 16) / 2` with no floor. On a single-core
machine this divides to `0`, and `Executors.newWorkStealingPool(0)` throws
`IllegalArgumentException` — this is exactly what happened on first run
here. Patched to clamp the result to a minimum of 1 thread. One line
changed; nothing else touched.

## Verified

- `java -jar d8.jar --help` prints D8's real usage text.
- Compiled a plain `Hello.java`, dexed it, confirmed the output is a
  genuine `Dalvik dex file version 035` containing the expected class/method
  via independent parsing (androguard).
- Compiled a class referencing real Android APIs (`AlertDialog`, `Intent`,
  `ContentResolver`, using `--lib android.jar`), dexed it successfully,
  confirmed all three resulting classes (including anonymous inner classes)
  parse correctly.

## Usage

```bash
# Plain Java, no Android APIs:
java -jar d8.jar --output out/ MyClass.class

# Java referencing Android APIs, needs the SDK stub jar as --lib:
java -jar d8.jar --lib android.jar --output out/ MyActivity.class

# A whole jar of classes:
java -jar d8.jar --output out/ my-classes.jar
```

`--output` must point to an existing directory or a `.zip`/`.jar` path —
create the directory first if it doesn't exist yet.
