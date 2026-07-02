---
name: gradle-convention-plugin
description: Creates and maintains Gradle Convention Plugins inside a build-logic included build for Android multi-module projects. Use when user asks to "create a convention plugin", "add a new plugin to build-logic", "share build config across modules", "reduce build.gradle duplication", or "centralize Gradle config".
---

# Gradle Convention Plugin

Convention Plugins live in `build-logic/` and centralize all shared build config. Each module's `build.gradle.kts` becomes a short list of plugin aliases + its own `namespace`.

**Problem**: Each module duplicates 100+ lines of SDK versions, compiler options, and test deps.  
**Solution**: Write config once in a plugin, apply it everywhere.

---

## Full Project Structure

```
root/
├── build-logic/
│   ├── settings.gradle.kts              ← declares build-logic + shares version catalog
│   └── convention/
│       ├── build.gradle.kts             ← kotlin-dsl + AGP/Kotlin deps + plugin registrations
│       └── src/main/kotlin/
│           ├── extensions/
│           │   ├── ProjectExtension.kt          ← typed AGP accessors + libs catalog
│           │   └── DependencyHandlerExtension.kt ← string constants for dependency scopes
│           ├── AndroidLibraryConventionPlugin.kt
│           ├── AndroidApplicationConventionPlugin.kt
│           ├── ComposeConventionPlugin.kt
│           └── UnitTestConventionPlugin.kt
├── gradle/
│   └── libs.versions.toml
└── settings.gradle.kts                  ← includeBuild("build-logic")
```

---

## Step 1 — Wire build-logic

### `settings.gradle.kts` (root)

```kotlin
pluginManagement {
    includeBuild("build-logic")
    repositories {
        google()
        mavenCentral()
        gradlePluginPortal()
    }
}
```

### `build-logic/settings.gradle.kts`

```kotlin
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
    }
    versionCatalogs {
        create("libs") {
            from(files("../gradle/libs.versions.toml"))  // single source of truth
        }
    }
}

rootProject.name = "build-logic"
include(":convention")
```

---

## Step 2 — `build-logic/convention/build.gradle.kts`

```kotlin
plugins {
    `kotlin-dsl`
}

repositories {
    google()
    mavenCentral()
}

dependencies {
    compileOnly(libs.android.tools.build.gradle)
    compileOnly(libs.kotlin.gradle)
}

gradlePlugin {
    plugins {
        register("androidLibrary") {
            id = "my.android.library"
            implementationClass = "AndroidLibraryConventionPlugin"
        }
        register("androidApplication") {
            id = "my.android.application"
            implementationClass = "AndroidApplicationConventionPlugin"
        }
        register("compose") {
            id = "my.compose"
            implementationClass = "ComposeConventionPlugin"
        }
        register("unitTest") {
            id = "my.test.unit"
            implementationClass = "UnitTestConventionPlugin"
        }
    }
}
```

---

## Step 3 — Extension helpers (`internal` — only used inside build-logic)

### `extensions/ProjectExtension.kt`

```kotlin
package extensions

import com.android.build.api.dsl.ApplicationExtension
import com.android.build.api.dsl.CommonExtension
import com.android.build.api.dsl.LibraryExtension
import org.gradle.api.JavaVersion
import org.gradle.api.Project
import org.gradle.api.artifacts.VersionCatalog
import org.gradle.api.artifacts.VersionCatalogsExtension
import org.gradle.api.plugins.JavaPluginExtension
import org.gradle.kotlin.dsl.configure
import org.gradle.kotlin.dsl.getByType
import org.gradle.kotlin.dsl.withType
import org.jetbrains.kotlin.gradle.dsl.JvmTarget
import org.jetbrains.kotlin.gradle.tasks.KotlinCompile

internal val javaVersion = JavaVersion.VERSION_21

internal val Project.libs: VersionCatalog
    get() = extensions.getByType<VersionCatalogsExtension>().named("libs")

internal fun Project.libraryExtension(): LibraryExtension =
    extensions.getByType(LibraryExtension::class)

internal fun Project.applicationExtension(): ApplicationExtension =
    extensions.getByType(ApplicationExtension::class)

internal fun Project.commonExtension(): CommonExtension<*, *, *, *, *, *> =
    extensions.getByType(CommonExtension::class)

internal fun Project.configureKotlinJvm() {
    extensions.configure<JavaPluginExtension> {
        sourceCompatibility = javaVersion
        targetCompatibility = javaVersion
    }
    tasks.withType<KotlinCompile>().configureEach {
        compilerOptions.jvmTarget.set(JvmTarget.JVM_21)
    }
}
```

### `extensions/DependencyHandlerExtension.kt`

```kotlin
package extensions

import org.gradle.api.artifacts.dsl.DependencyHandler

internal val DependencyHandler.implementation: String get() = "implementation"
internal val DependencyHandler.testImplementation: String get() = "testImplementation"
internal val DependencyHandler.androidTestImplementation: String get() = "androidTestImplementation"
internal val DependencyHandler.debugImplementation: String get() = "debugImplementation"
internal val DependencyHandler.ksp: String get() = "ksp"
```

---

## Step 4 — Plugin class anatomy

Every plugin follows the same three-method structure:

```kotlin
class MyConventionPlugin : Plugin<Project> {

    override fun apply(project: Project) {
        applyPlugins(project)       // 1. apply Gradle/AGP plugins this plugin depends on
        setProjectConfig(project)   // 2. configure Android/Kotlin extensions
        applyDependencies(project)  // 3. add required dependencies
    }

    private fun applyPlugins(project: Project) { ... }
    private fun setProjectConfig(project: Project) { ... }
    private fun applyDependencies(project: Project) { ... }
}
```

> Not every plugin needs all three — a simple tool plugin (serialization, collections) may only need `applyPlugins` + `applyDependencies`.

---

## Plugin Examples

### `AndroidLibraryConventionPlugin.kt`

```kotlin
import extensions.configureKotlinJvm
import extensions.implementation
import extensions.javaVersion
import extensions.libraryExtension
import extensions.libs
import org.gradle.api.Plugin
import org.gradle.api.Project
import org.gradle.kotlin.dsl.apply
import org.gradle.kotlin.dsl.dependencies

class AndroidLibraryConventionPlugin : Plugin<Project> {

    override fun apply(project: Project) {
        applyPlugins(project)
        setProjectConfig(project)
        applyDependencies(project)
    }

    private fun applyPlugins(project: Project) {
        project.apply(plugin = "com.android.library")
    }

    private fun setProjectConfig(project: Project) {
        project.libraryExtension().apply {
            compileSdk = project.libs.findVersion("compile.sdk").get().toString().toInt()
            defaultConfig {
                minSdk = project.libs.findVersion("min.sdk").get().toString().toInt()
            }
            compileOptions {
                sourceCompatibility = javaVersion
                targetCompatibility = javaVersion
            }
            buildTypes {
                release {
                    proguardFiles(getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro")
                }
            }
            packaging {
                resources { excludes += "/META-INF/{AL2.0,LGPL2.1}" }
            }
        }
        project.configureKotlinJvm()
    }

    private fun applyDependencies(project: Project) {
        project.dependencies {
            add(implementation, project.libs.findLibrary("androidx.core.ktx").get())
        }
    }
}
```

### `ComposeConventionPlugin.kt`

```kotlin
import extensions.androidTestImplementation
import extensions.commonExtension
import extensions.debugImplementation
import extensions.implementation
import extensions.libs
import org.gradle.api.Plugin
import org.gradle.api.Project
import org.gradle.kotlin.dsl.apply
import org.gradle.kotlin.dsl.dependencies

class ComposeConventionPlugin : Plugin<Project> {

    override fun apply(project: Project) {
        applyPlugins(project)
        setProjectConfig(project)
        applyDependencies(project)
    }

    private fun applyPlugins(project: Project) {
        // Read plugin ID from catalog — avoids hardcoding
        val pluginId = project.libs.findPlugin("kotlin.compose").get().get().pluginId
        project.apply(plugin = pluginId)
    }

    private fun setProjectConfig(project: Project) {
        project.commonExtension().buildFeatures.compose = true
    }

    private fun applyDependencies(project: Project) {
        project.dependencies {
            val bom = project.libs.findLibrary("androidx.compose.bom").get()
            add(implementation, platform(bom))
            add(implementation, project.libs.findLibrary("androidx.compose.ui").get())
            add(implementation, project.libs.findLibrary("androidx.compose.material3").get())
            add(debugImplementation, project.libs.findLibrary("androidx.compose.ui.tooling").get())
            add(androidTestImplementation, platform(bom))
        }
    }
}
```

### `UnitTestConventionPlugin.kt`

```kotlin
import extensions.libs
import extensions.testImplementation
import org.gradle.api.Plugin
import org.gradle.api.Project
import org.gradle.api.tasks.testing.Test
import org.gradle.kotlin.dsl.dependencies
import org.gradle.kotlin.dsl.withType

class UnitTestConventionPlugin : Plugin<Project> {

    override fun apply(project: Project) {
        applyDependencies(project)
        project.tasks.withType<Test>().configureEach { useJUnitPlatform() }
    }

    private fun applyDependencies(project: Project) {
        project.dependencies {
            add(testImplementation, project.libs.findLibrary("test.junit").get())
            add(testImplementation, project.libs.findLibrary("test.mockk").get())
            add(testImplementation, project.libs.findLibrary("test.coroutine").get())
            add(testImplementation, project.libs.findLibrary("turbine").get())
        }
    }
}
```

### Minimal tool plugin (no AGP — serialization, immutable collections, etc.)

```kotlin
import extensions.implementation
import extensions.libs
import org.gradle.api.Plugin
import org.gradle.api.Project
import org.gradle.kotlin.dsl.apply
import org.gradle.kotlin.dsl.dependencies

class SerializationConventionPlugin : Plugin<Project> {
    override fun apply(project: Project) {
        val pluginId = project.libs.findPlugin("kotlin.serialization").get().get().pluginId
        project.apply(plugin = pluginId)
        project.dependencies {
            add(implementation, project.libs.findLibrary("jetbrains.kotlin.serialization").get())
        }
    }
}
```

---

## Step 5 — Declare aliases in `libs.versions.toml`

Convention plugins from an included build have no real version — use `version = "unspecified"`:

```toml
[plugins]
# Third-party plugins (real versions)
android-application   = { id = "com.android.application",                   version.ref = "agp" }
kotlin-compose        = { id = "org.jetbrains.kotlin.plugin.compose",       version.ref = "kotlin" }
kotlin-serialization  = { id = "org.jetbrains.kotlin.plugin.serialization", version.ref = "kotlin" }

# Convention plugins — MUST use version = "unspecified"
my-android-library    = { id = "my.android.library",      version = "unspecified" }
my-android-app        = { id = "my.android.application",  version = "unspecified" }
my-compose            = { id = "my.compose",              version = "unspecified" }
my-unit-test          = { id = "my.test.unit",            version = "unspecified" }
```

---

## Step 6 — Apply in a module

```kotlin
// feature/my-feature/build.gradle.kts
plugins {
    alias(libs.plugins.my.android.library)
    alias(libs.plugins.my.compose)
    alias(libs.plugins.my.unit.test)
}

android {
    namespace = "com.example.feature.myfeature"
}

dependencies {
    implementation(project(":component:my-component"))
}
```

---

## When to Create a New Plugin

- Copy-pasting 3+ lines across multiple `build.gradle.kts`
- New tool added project-wide (Detekt, Jacoco, Paparazzi)
- New module type introduced (e.g., `integration`, `common-ui`)
- Existing plugin has two unrelated concerns → split into two

**One plugin per concern**: Compose, UnitTest, and Detekt are separate plugins so each module opts in independently.

---

## Rules

| ✅ Do | ❌ Don't |
|---|---|
| Read versions via `libs.findVersion("key").get().toString()` | Hardcode `compileSdk = 35` in plugin code |
| Read libraries via `libs.findLibrary("key").get()` | Use string coordinates `"group:artifact:1.0"` |
| Read plugin IDs via `libs.findPlugin("key").get().get().pluginId` | Hardcode plugin IDs as strings |
| `version = "unspecified"` for convention plugin aliases | Omit the version field (causes resolution error) |
| `internal` for all extension helpers | Expose extension helpers as `public` |
| One plugin per concern | Bundle unrelated concerns in one plugin |

---

## Checklist: New Convention Plugin

- [ ] Plugin class in `build-logic/convention/src/main/kotlin/`
- [ ] Follows `applyPlugins / setProjectConfig / applyDependencies` structure
- [ ] All versions/libraries/plugins read via `project.libs.find*` — no hardcoded strings
- [ ] Extension helpers used (`libraryExtension()`, `commonExtension()`, scope constants)
- [ ] Registered in `build-logic/convention/build.gradle.kts` with a stable ID
- [ ] Alias declared in `libs.versions.toml` with `version = "unspecified"`
- [ ] Applied in at least one module — `./gradlew assembleDebug` passes

---

## References

- [Gradle Convention Plugins Guide](https://docs.gradle.org/current/samples/sample_convention_plugins.html)
- [Now in Android — build-logic reference](https://github.com/android/nowinandroid/tree/main/build-logic)
- [Gradle Plugin Development](https://docs.gradle.org/current/userguide/custom_plugins.html)
- [Version Catalogs — accessing from plugins](https://docs.gradle.org/current/userguide/version_catalogs.html)
- [Android Gradle Plugin API](https://developer.android.com/build/extend-agp)

