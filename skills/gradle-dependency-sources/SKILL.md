---
name: gradle-dependency-sources
description: >
  Inspect Gradle dependency sources to learn APIs of project dependencies.
  Use when you need to understand how to use a specific library or dependency
  in a Gradle-based Java/Kotlin project, especially when your knowledge
  may be outdated for the dependency version used by the project.
---

# Gradle Dependency Sources

This skill helps you inspect the actual source code of Gradle project dependencies so you can accurately use their APIs,
even when your training data may be outdated for the specific version in use.

## When to use

- You need to use a dependency API but are unsure of the exact method signatures, class names, or patterns for the
  version in the project
- The user asks you to use a library you're uncertain about
- You're working with an internal/company dependency you have no training data for
- You want to verify your knowledge of a dependency's API before writing code

## Workflow

Follow these steps in order. Skip steps when possible (e.g., if the GAV is already known, skip discovery steps).

### Step 0: Check extracted sources cache FIRST

If you already know the dependency's group, artifact, and version (GAV), check whether sources have already been
extracted before running any Gradle commands.

Check whether sources have already been extracted to the project-local cache at
`.gradle/jarsources/<group>/<artifact>/<version>/` (dots in group name as-is, e.g. `org.springframework`).

**If the directory exists and contains `.java` or `.kt` files, skip directly to Step 5.**

### Step 1: Discover project structure

Only needed if you don't know the project's subproject layout:

```bash
./gradlew -q projects
```

This lists all subprojects. Use this to determine whether the dependency is declared on the root project or on a
specific subproject.

- Root project task path: `dependencies`
- Subproject task path example: `:app:dependencies`

If `./gradlew` is not found, try `gradle` directly.

### Step 2: List dependencies to find exact GAV

Run the dependencies task for the relevant project (root or subproject):

```bash
# Root project
./gradlew -q dependencies --configuration runtimeClasspath

# Subproject example ("app")
./gradlew -q :app:dependencies --configuration runtimeClasspath
```

Replace `app` with the actual subproject path when needed.

This prints the dependency tree. Find the dependency of interest and note its exact group, artifact, and version. The
output format is:

```
+--- org.springframework:spring-jdbc:7.0.2
```

If `runtimeClasspath` doesn't exist (e.g., non-JVM plugin), try `compileClasspath` instead.

Troubleshooting: if both are unavailable, use the configuration names reported by Gradle and rerun Step 2 with the
most relevant JVM classpath-style configuration for that project.

### Step 3: Check cache again

Now that you know the exact GAV from Step 2, check the cache again at
`.gradle/jarsources/<group>/<artifact>/<version>/`. If sources exist, skip to Step 5.

### Step 4: Download and extract sources

Use the bundled init script to download and extract the dependency sources. The init script is located relative to this
skill file at `scripts/downloadSources.gradle`.

Run:

```bash
./gradlew --init-script <path-to-skill>/scripts/downloadSources.gradle \
  --console plain --warning-mode none --no-problems-report :downloadSourceJar \
  -Pdep.notation=<group>:<artifact>:<version> \
  -Pdep.target.dir=.gradle/jarsources/<group>/<artifact>/<version>
```

Replace:

- `<path-to-skill>` with the absolute path to the directory from which this skill was loaded (the directory containing
  `SKILL.md`)
- `<group>:<artifact>:<version>` with the dependency GAV (do NOT include a classifier - the script adds `:sources`
  automatically)

**Expected output on success:**

```
SOURCES_EXTRACTED:/path/to/extracted/sources
PACKAGES:
  com.example.api
  com.example.api.model
  com.example.internal
```

The `PACKAGES` section lists Java/Kotlin packages declared in the extracted source files, one per line. Use this to
quickly identify which packages are relevant before reading individual files.

**If the sources JAR is not available** (not all artifacts publish sources), the script will output an error. In that
case:

- Inform the user that sources are not available for this artifact
- Consider using `javap` on the regular JAR from Gradle's cache to inspect class signatures as a fallback

### Step 5: Read relevant source files

Now read the extracted sources to learn the API. The sources are in
`.gradle/jarsources/<group>/<artifact>/<version>/`. Be selective - do NOT read every file. Focus on:

1. **The specific class or interface** the user is working with
2. **Public methods and their signatures** - parameters, return types, exceptions
3. **Package-info.java** if present - often contains package-level documentation
4. **Key abstract classes or interfaces** that define the API contract

Locate the files you need (by class name or glob pattern), then read only the relevant ones.

## Important notes

- **Notation format**: Always pass `group:artifact:version` (3 parts). The init script handles the `:sources`
  classifier.
- **Caching**: Extracted sources persist in `<project-root>/.gradle/jarsources/`. Same version is never re-downloaded.
  This directory is typically already gitignored since `.gradle/` is a standard Gradle cache location.
- **SNAPSHOT versions**: The cache does not auto-invalidate. If a dependency uses a `-SNAPSHOT` version, the cached
  sources may be stale. Delete the corresponding `.gradle/jarsources/<group>/<artifact>/<version>/` directory and
  re-run Step 4 to refresh.
- **Safety**: The init script uses a detached configuration and never modifies the project's build files or
  configurations.
- **Multi-module**: Always verify you're querying the correct subproject for dependencies.
- **Be selective**: Read only the source files relevant to the task. Don't dump entire packages into context.
