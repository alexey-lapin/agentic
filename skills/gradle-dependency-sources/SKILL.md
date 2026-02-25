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

Determine `GRADLE_USER_HOME`:

```bash
echo "${GRADLE_USER_HOME:-$HOME/.gradle}"
```

Then check for extracted sources:

```bash
ls "${GRADLE_USER_HOME:-$HOME/.gradle}/jarsources/<group>/<artifact>/<version>/"
```

Replace `<group>`, `<artifact>`, `<version>` with the actual values, using dots in group name as-is (e.g.,
`org.springframework`).

**If the directory exists and contains `.java` or `.kt` files, skip directly to Step 5.**

### Step 1: Discover project structure

Only needed if you don't know the project's subproject layout:

```bash
./gradlew -q projects
```

This lists all subprojects. Use this to determine the correct `:<project-path>:` prefix for Gradle tasks. For the root
project, use `:` (empty path). For a subproject named `app`, use `:app`.

If `./gradlew` is not found, try `gradle` directly.

### Step 2: List dependencies to find exact GAV

```bash
./gradlew -q :<project-path>:dependencies --configuration runtimeClasspath
```

This prints the dependency tree. Find the dependency of interest and note its exact group, artifact, and version. The
output format is:

```
+--- org.springframework:spring-jdbc:7.0.2
```

If `runtimeClasspath` doesn't exist (e.g., non-JVM plugin), try `compileClasspath` instead.

### Step 3: Check cache again

Now that you know the exact GAV from Step 2, check the cache again:

```bash
ls "${GRADLE_USER_HOME:-$HOME/.gradle}/jarsources/<group>/<artifact>/<version>/"
```

If sources exist, skip to Step 5.

### Step 4: Download and extract sources

Use the bundled init script to download and extract the dependency sources. The init script is located relative to this
skill file at `scripts/downloadSources.gradle`.

Determine the path to the init script. It is in the same directory as this skill file, under
`scripts/downloadSources.gradle`. For example, if this skill is installed at
`~/.config/opencode/skills/gradle-dependency-sources/SKILL.md`, the init script is at
`~/.config/opencode/skills/gradle-dependency-sources/scripts/downloadSources.gradle`.

Run:

```bash
GRADLE_USER_HOME="${GRADLE_USER_HOME:-$HOME/.gradle}"

./gradlew --init-script <path-to-skill>/scripts/downloadSources.gradle \
  -q downloadSourceJar \
  -Pdep.notation=<group>:<artifact>:<version> \
  "-Pdep.target.dir=$GRADLE_USER_HOME/jarsources/<group>/<artifact>/<version>"
```

Replace:

- `<path-to-skill>` with the actual path to this skill's directory
- `<group>:<artifact>:<version>` with the dependency GAV (do NOT include a classifier - the script adds `:sources`
  automatically)

**Expected output on success:**

```
SOURCES_EXTRACTED:/path/to/extracted/sources
```

**If the sources JAR is not available** (not all artifacts publish sources), the script will output an error. In that
case:

- Inform the user that sources are not available for this artifact
- Consider using `javap` on the regular JAR from Gradle's cache to inspect class signatures as a fallback

### Step 5: Read relevant source files

Now read the extracted sources to learn the API. Be selective - do NOT read every file. Focus on:

1. **The specific class or interface** the user is working with
2. **Public methods and their signatures** - parameters, return types, exceptions
3. **Package-info.java** if present - often contains package-level documentation
4. **Key abstract classes or interfaces** that define the API contract

Use glob/find to locate specific files:

```bash
find "${GRADLE_USER_HOME:-$HOME/.gradle}/jarsources/<group>/<artifact>/<version>/" -name "ClassName.java"
```

Or browse the package structure:

```bash
find "${GRADLE_USER_HOME:-$HOME/.gradle}/jarsources/<group>/<artifact>/<version>/" -name "*.java" | head -50
```

Then read the specific files you need to understand the API.

## Important notes

- **Notation format**: Always pass `group:artifact:version` (3 parts). The init script handles the `:sources`
  classifier.
- **Caching**: Extracted sources persist in `$GRADLE_USER_HOME/jarsources/`. Same version is never re-downloaded.
- **Safety**: The init script uses a detached configuration and never modifies the project's build files or
  configurations.
- **Multi-module**: Always verify you're querying the correct subproject for dependencies.
- **Be selective**: Read only the source files relevant to the task. Don't dump entire packages into context.
