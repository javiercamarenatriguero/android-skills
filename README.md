# Android Skills Library

A collection of **project-agnostic** skills for Android development. These skills contain no references to specific companies, products, or internal libraries — any Android developer can use them.

---

## Skills Catalog

| Skill | Description | Key Topics |
|-------|-------------|------------|
| [android-modularization](./android-modularization/SKILL.md) | Multi-module architecture with 4-layer separation | App / Feature / Component / Common, dependency rules, integration modules |
| [mvi-editor](./mvi-editor/SKILL.md) | Model-View-Intent pattern from scratch | ViewState, ViewEvent, SideEffect, ViewModel, Screen wiring, navigation side effects |
| [compose-editor](./compose-editor/SKILL.md) | Jetpack Compose UI development patterns | State hoisting, side effects, accessibility, lazy lists, theming, best practices |
| [compose-performance-auditor](./compose-performance-auditor/SKILL.md) | Compose runtime performance review | Recomposition smells, stability, `derivedStateOf`, `remember`, phase deferral |
| [kotlin-coroutines](./kotlin-coroutines/SKILL.md) | Coroutines and Flow best practices | Structured concurrency, dispatchers, StateFlow, SharedFlow, error handling, testing |
| [kotlin-convention](./kotlin-convention/SKILL.md) | Kotlin idioms and code conventions | Naming, null safety, sealed interfaces, value classes, collections, error handling |
| [gradle-convention-plugin](./gradle-convention-plugin/SKILL.md) | Creating and using Gradle Convention Plugins | Plugin authoring, shared config, module templates, buildSrc vs included builds |
| [gradle-configuration](./gradle-configuration/SKILL.md) | Gradle project configuration | Version catalogs, dependency scopes, conflict resolution, build performance |
| [koin-editor](./koin-editor/SKILL.md) | Dependency injection with Koin | Module definition, scopes, qualifiers, testing overrides, multi-module setup |
| [android-unit-test-editor](./android-unit-test-editor/SKILL.md) | Unit testing for Android with MockK | GIVEN/WHEN/THEN naming, UseCase/ViewModel/Repository templates, Turbine, coroutines |
| [paparazzi-editor](./paparazzi-editor/SKILL.md) | Snapshot testing with Paparazzi | Setup, multi-state tests, dark mode, device configs, CI integration |
| [github-action-editor](./github-action-editor/SKILL.md) | GitHub Actions for Android CI/CD | Workflow structure, caching, matrix builds, secrets, artifact uploads, optimization |
| [android-permissions-editor](./android-permissions-editor/SKILL.md) | Android runtime permissions | PermissionState model, rationale dialogs, permanently denied, Compose integration, MVI wiring |

---

## How to Use

Each skill folder contains a `SKILL.md` with:
- **Patterns and templates** — copy-paste ready code
- **Do/Don't tables** — quick decision guides
- **Checklists** — before-you-commit reminders
- **References** — links to official documentation

### Invoking a Skill

When using a GitHub Copilot agent:
```
skill: compose-performance-auditor
skill: android-unit-test-editor
skill: github-action-editor
```

Or reference directly when working on a specific task:
> "Follow the patterns in `koin-editor/SKILL.md` when registering the new repository."

---

## Design Principles

1. **Zero project references** — No company names, product names, or internal library names
2. **Generic by default** — Works with any Android project, any team
3. **Opinionated but justified** — Every recommendation includes a "why"
4. **Actionable** — Templates and examples over abstract advice
5. **Current** — Based on modern Android (Kotlin 2.x, AGP 8.x, Compose stable)
