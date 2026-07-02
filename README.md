# Android Skills Library

A collection of **project-agnostic** skills for Android development. These skills contain no references to specific companies, products, or internal libraries — any Android developer can use them.

---

## Skills Catalog

| Skill                                                                 | Description                                       | Key Topics                                                                                         |
|-----------------------------------------------------------------------|---------------------------------------------------|----------------------------------------------------------------------------------------------------|
| [android-modularization](./android-modularization/SKILL.md)           | Multi-module architecture with 4-layer separation | App / Feature / Component / Common, dependency rules, integration modules                          |
| [mvi-editor](./mvi-editor/SKILL.md)                                   | MVI pattern — library-free, self-contained        | ViewState, ViewEvent, SideEffect, ViewModel, Screen wiring, navigation side effects                |
| [compose-editor](./compose-editor/SKILL.md)                           | Idiomatic Jetpack Compose UI patterns             | State hoisting, side effects, accessibility, lazy lists, theming, best practices                   |
| [compose-performance-auditor](./compose-performance-auditor/SKILL.md) | Compose runtime performance audit and fix         | Recomposition smells, stability, `derivedStateOf`, `remember`, phase deferral                      |
| [kotlin-coroutines](./kotlin-coroutines/SKILL.md)                     | Correct, performant Coroutines and Flow           | Structured concurrency, dispatchers, StateFlow, SharedFlow, error handling, testing                |
| [kotlin-convention](./kotlin-convention/SKILL.md)                     | Idiomatic Kotlin for Android                      | Naming, null safety, sealed interfaces, value classes, collections, error handling                 |
| [gradle-convention-plugin](./gradle-convention-plugin/SKILL.md)       | Gradle Convention Plugins for multi-module apps   | Plugin authoring, shared config, module templates, included builds                                 |
| [gradle-configuration](./gradle-configuration/SKILL.md)               | Gradle build configuration and dependency mgmt    | Version catalogs, dependency scopes, conflict resolution, build performance                        |
| [koin-editor](./koin-editor/SKILL.md)                                 | Koin dependency injection for Android/KMP         | Module definition, scopes, qualifiers, testing overrides, multi-module setup                       |
| [android-unit-test-editor](./android-unit-test-editor/SKILL.md)       | Unit tests for Android with MockK                 | GIVEN/WHEN/THEN naming, UseCase/ViewModel/Repository templates, Turbine, coroutines                |
| [paparazzi-editor](./paparazzi-editor/SKILL.md)                       | Snapshot tests with Paparazzi                     | Setup, multi-state tests, dark mode, device configs, CI integration                                |
| [github-action-editor](./github-action-editor/SKILL.md)               | GitHub Actions CI/CD for Android                  | Workflow structure, caching, matrix builds, secrets, artifact uploads, optimization                |
| [android-permissions-editor](./android-permissions-editor/SKILL.md)   | Android runtime permissions (multi-module)        | PermissionState model, rationale dialogs, permanently denied, Compose integration, MVI wiring      |
| [compose-navigation3](./compose-navigation3/SKILL.md)                 | Type-safe navigation with Jetpack Navigation 3    | NavKey routes, NavDisplay, navigator abstractions, ResultStore, cross-screen results, transitions  |
| [review-owasp-security](./review-owasp-security/SKILL.md)             | OWASP MASVS security review                       | Data storage, networking, authentication, deep links, permissions, secrets — PASS/WARN/FAIL report |

---

## How to Use

Each skill folder contains a `SKILL.md` with:
- **Patterns and templates** — copy-paste ready code
- **Do/Don't tables** — quick decision guides
- **Checklists** — before-you-commit reminders
- **References** — links to official documentation

### Installation

Copy the skill folder to your project's `.claude/skills/` directory so the agent picks it up automatically:

```bash
# Install a skill in your project
cp -r android-unit-test-editor /your-project/.claude/skills/

# Or install all skills at once
cp -r */ /your-project/.claude/skills/
```

For personal use across all projects, copy to `~/.claude/skills/` instead.

### Skill Discovery

Claude activates a skill automatically when the task matches the skill's description trigger phrases. For explicit invocation:

```
Use the android-unit-test-editor skill to write tests for the OrderRepository.
Apply compose-performance-auditor to this screen code.
Follow the mvi-editor skill when creating the new ProfileScreen.
```

> **Note**: Skills are snapshotted at session start. If you add or edit a skill, restart the session.

### How loading works

Skills use a three-level loading model to stay token-efficient:

1. **Level 1 (always loaded)** — only `name` + `description` from each skill (~100 tokens each)
2. **Level 2 (on trigger)** — full `SKILL.md` body, read when the skill becomes relevant
3. **Level 3 (on demand)** — any referenced files, loaded only when that section is needed

This means you can install all 15 skills with a negligible context cost at idle.

### Design Principles

1. **Zero project references** — No company names, product names, or internal library names
2. **Generic by default** — Works with any Android project, any team
3. **Opinionated but justified** — Every recommendation includes a "why"
4. **Actionable** — Templates and examples over abstract advice
5. **Current** — Based on modern Android (Kotlin 2.x, AGP 8.x, Compose stable)
