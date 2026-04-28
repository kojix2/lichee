# DIFF.md

## Purpose

This file records the functional and operational differences between the original LICHeE distribution and this fork.

The main purpose of this fork is to make LICHeE easier to build and run on modern Java environments, especially in headless environments where no X11 server is available. However, the patch is not limited to Java compatibility. It also changes the build model, runtime invocation, CLI behavior, and some DOT / visualization output details.

## Summary

Compared with the original LICHeE, this fork introduces the following major changes:

1. Adds a source build path using `make lichee.jar`.
2. Changes the launcher from `java -jar lichee.jar` to explicit classpath execution.
3. Treats `release/lichee.jar` as a generated artifact instead of a tracked binary.
4. Adds GitHub Actions CI for Java 17 and Java 21 on Linux and macOS.
5. Removes an X11-dependent screen-resolution lookup from DOT generation.
6. Fixes `-dotFile` handling for relative output paths.
7. Changes `-h` / `--help` to exit immediately after printing help.
8. Changes internal node labels in tree visualization / DOT output.
9. Performs small type and spelling cleanups in source code and documentation.

The core lineage inference algorithm is not intentionally changed.

## Compatibility impact

### Java compatibility

The original distribution relied on a prebuilt `release/lichee.jar` and a launcher that executed:

```sh
java -jar lichee.jar "$@"
```

This fork instead builds `release/lichee.jar` from source and runs LICHeE with an explicit classpath:

```sh
java -cp "$SCRIPT_DIR/lichee.jar:$SCRIPT_DIR/../lib/*" lineage.LineageEngine "$@"
```

This makes execution more robust with current Java installations, because dependency jars under `LICHeE/lib/` are placed directly on the classpath.

However, this also means that `release/lichee.jar` is no longer intended to be a self-contained executable jar. Users should run LICHeE through `LICHeE/release/lichee`, not by invoking `java -jar release/lichee.jar` directly.

### Build and distribution compatibility

The original repository included a prebuilt `LICHeE/release/lichee.jar`.

This fork renames the original jar to `LICHeE/release/lichee_orginal.jar` and ignores newly generated jar files through `.gitignore`.

As a result, a fresh checkout requires a build step:

```sh
cd LICHeE
make lichee.jar
```

This is a deliberate operational change. It reduces reliance on an old binary artifact, but it also means the repository is no longer immediately runnable unless the jar has been built.

## Detailed changes

### 1. Continuous integration

A new GitHub Actions workflow is added at:

```text
.github/workflows/ci.yml
```

The workflow builds and smoke-tests LICHeE on:

- Ubuntu latest
- macOS latest
- Java 17
- Java 21

The smoke test runs LICHeE in CLI mode without GUI:

```sh
./release/lichee \
  -build \
  -i data/ccRCC/RK26.txt \
  -maxVAFAbsent 0.005 \
  -minVAFPresent 0.005 \
  -n 0 \
  -dotFile "$RUNNER_TEMP/rk26.dot"
```

It then checks that the DOT output file exists and is non-empty.

Functional impact: this does not change runtime behavior directly, but it defines the supported build/test target as Java 17 and Java 21 on Linux/macOS.

### 2. `.gitignore`

A new `.gitignore` ignores:

```text
build/
*.jar
```

Functional impact: generated build products are no longer tracked. This reinforces the new build-from-source model.

### 3. Build system

A new `LICHeE/Makefile` adds a `lichee.jar` target.

The build process:

1. Removes the old `build/` directory.
2. Creates a fresh `build/` directory.
3. Compiles `src/lineage/LineageEngine.java` with all jars under `lib/` on the classpath.
4. Packages compiled classes into `release/lichee.jar`.

Functional impact: users and CI can rebuild the jar using a simple command. This is an operational improvement, not an algorithmic change.

### 4. Launcher script

The launcher at `LICHeE/release/lichee` is changed.

Original behavior:

```sh
java -jar lichee.jar "$@"
```

New behavior:

```sh
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"

if [ ! -f "$SCRIPT_DIR/lichee.jar" ]; then
    echo "error: $SCRIPT_DIR/lichee.jar not found. build it first: (cd $SCRIPT_DIR/.. && make lichee.jar)" >&2
    exit 1
fi

java -cp "$SCRIPT_DIR/lichee.jar:$SCRIPT_DIR/../lib/*" lineage.LineageEngine "$@"
```

Functional impact:

- The script can be run from any current working directory because it resolves its own location.
- Missing `lichee.jar` now produces a clear error message.
- Dependencies are loaded from `../lib/*` instead of relying on jar manifest behavior.
- Direct `java -jar release/lichee.jar` is no longer the expected execution path.

This is one of the most important behavioral changes in the patch.

### 5. Original jar handling

The original jar is renamed:

```text
LICHeE/release/lichee.jar -> LICHeE/release/lichee_orginal.jar
```

Note: the filename contains `orginal`, not `original`.

Functional impact: the old binary is preserved but no longer used by the new launcher. The active jar is expected to be generated by `make lichee.jar`.

### 6. DOT output path handling

In `LineageEngine.java`, parent directory resolution for `-dotFile` changes from:

```java
String parentDir = new File(args.outputDOTFileName).getParent();
```

to:

```java
String parentDir = new File(args.outputDOTFileName).getAbsoluteFile().getParent();
```

Functional impact: `-dotFile rk26.dot` now has a usable parent directory even when the user provides a filename without an explicit directory component.

This is a bug fix for relative DOT output paths.

### 7. Help option behavior

The `-h` / `--help` handling is moved earlier in argument parsing.

New behavior:

```java
if(cmdLine.hasOption("h")) {
    hf.printHelp("lichee", options);
    System.exit(0);
}
```

Functional impact:

- `lichee -h` prints help and exits successfully.
- The program no longer continues into later setup code after printing help.
- Exit status is now `0` for help.

This is a CLI behavior fix.

### 8. Option comparator typo cleanup

The internal class name is changed from:

```java
OptionComarator
```

to:

```java
OptionComparator
```

Functional impact: none expected. This is a spelling / readability cleanup.

### 9. Parameter type cleanup

In `Parameters.java`, two threshold fields are changed from `double` to `int`:

```java
MIN_GROUP_PROFILE_SUPPORT: double -> int
MIN_ROBUST_CLUSTER_SUPPORT: double -> int
```

Functional impact: none expected for normal CLI usage, because these values represent counts and are parsed as integers. This aligns the field types with their actual semantics.

Potential compatibility note: any external code modifying these fields as non-integer values would no longer compile, but that is not expected for normal LICHeE use.

### 10. Headless DOT generation

In `LineageDisplayConfig.java`, DOT node size attributes based on screen resolution are disabled:

```java
//s += " width=" + getNodeShape(v).getBounds().getWidth()/Toolkit.getDefaultToolkit().getScreenResolution() + "";
//s += " heigth=" + getNodeShape(v).getBounds().getHeight()/Toolkit.getDefaultToolkit().getScreenResolution() + "";
```

Functional impact:

- DOT generation no longer calls `Toolkit.getDefaultToolkit().getScreenResolution()`.
- This avoids failures in headless environments without an X11 server.
- DOT output may render with slightly different node sizing in Graphviz.

Note: the original `heigth` attribute appears to be misspelled and likely had no effect in Graphviz. The removal of `width` is the more relevant visual change.

This is a real functional change in output rendering, even though it does not affect lineage inference.

### 11. Internal node labels

Internal node labels change from showing only cluster size:

```java
return n.getSize() + "";
```

to showing node id and size:

```java
return "" + (nid - 1) + "(" + n.getSize() + ")";
```

Functional impact:

- Tree visualization and DOT labels are changed.
- Internal nodes now include a node-like identifier plus size.
- Existing visual outputs will not be textually identical to the original.

This does not change inferred trees, but it changes how they are displayed and exported.

### 12. README changes

`README.md` is rewritten and modernized.

Notable documentation changes:

- Adds a `Changes in this fork` section.
- Documents `make lichee.jar` as the build step.
- Updates system requirements from Java 1.6 to Java 17 or later.
- Reorganizes options into tables.
- Mentions that DOT generation no longer requires screen resolution parsing.
- Describes `release/lichee.jar` as a generated artifact.

Functional impact: documentation only, but it changes the stated supported Java version and expected workflow.

### 13. README_ORIG.md

A new `README_ORIG.md` preserves the original README content with a small note about the fork.

Functional impact: none. This is useful for reference but is not a complete diff document.

### 14. MANIFEST.MF and jar-in-jar loader classes

The patch adds:

```text
LICHeE/MANIFEST.MF
LICHeE/lib/org/eclipse/jdt/internal/jarinjarloader/*.class
```

These appear to support Eclipse-style jar-in-jar loading.

However, the new Makefile packages the jar with:

```sh
jar -cfe release/lichee.jar lineage.LineageEngine -C build .
```

and the launcher uses explicit classpath execution.

Therefore, these jar-in-jar additions do not appear to be used by the current build/run path.

Functional impact: probably none in the current launcher path. They may be leftovers from an alternative packaging attempt and should be reviewed. If they are not needed, removing them would make the patch cleaner.

## Algorithmic impact

No intentional changes were found in the main LICHeE inference pipeline:

- SSNV calling
- VAF / CP interpretation
- SSNV clustering
- constraint network construction
- lineage tree search
- scoring
- QP consistency check

The patch is primarily about build, runtime compatibility, CLI behavior, and output generation.

However, output files and visualizations may differ because of:

1. Changed internal node labels.
2. Removed DOT node width attribute.
3. Improved relative path handling for DOT output.

Therefore, exact textual or visual regression tests against old DOT files may fail even when the inferred tree is equivalent.

## User-visible changes

Users may notice the following differences:

### New required build step

Before running from a fresh checkout:

```sh
cd LICHeE
make lichee.jar
```

### Different execution expectation

Recommended:

```sh
./release/lichee ...
```

Not recommended / not guaranteed:

```sh
java -jar release/lichee.jar ...
```

### Better behavior in headless environments

DOT output can be generated without requiring an X11 display.

### More reliable `-dotFile` behavior

Relative DOT filenames should work more reliably.

### Help exits cleanly

```sh
./release/lichee -h
```

prints help and exits with status 0.

### Different node labels in visualization

Internal nodes are labeled with both an identifier and size rather than size only.

## Risk assessment

### Low risk

- CI addition
- README cleanup
- spelling fixes
- parameter type cleanup from `double` to `int`
- help option early exit
- relative `-dotFile` path fix

### Medium risk

- launcher change from `java -jar` to `java -cp`
- treating `lichee.jar` as a generated artifact
- depending on wildcard classpath behavior under `lib/*`

These are likely correct, but they change how users run and package the tool.

### Output compatibility risk

- removed DOT `width` attribute
- changed internal node labels

These can change DOT / Graphviz output even if the biological inference result is unchanged.

### Cleanup recommended

The added `MANIFEST.MF` and Eclipse jar-in-jar loader classes should be reviewed. They appear unused by the current Makefile and launcher. If the intended distribution model is explicit classpath execution, these files can probably be removed.

## Recommended description for the patch

A precise summary would be:

> Modernize LICHeE build and execution for current Java environments, add CI, make the jar a generated artifact, remove X11-dependent DOT generation behavior, and fix several CLI / DOT output issues.

A less accurate summary would be:

> Make LICHeE work with newer Java.

That description is incomplete because the patch also changes build/distribution behavior, DOT output behavior, help behavior, and visualization labels.

