---
name: compose-navigation3
description: Implements type-safe navigation using Jetpack Navigation 3 (Nav3) with Compose. Use when user asks to "add a new screen", "wire navigation", "pass results between screens", "set up Nav3", or "navigate to a destination". Covers NavKey routes, NavDisplay, navigator abstractions, and ResultStore.
---

# Compose Navigation 3

## Overview

Jetpack Navigation 3 (Nav3) replaces the `NavController` / `NavGraphBuilder` model with a **state-first** API:

| Nav2 concept | Nav3 replacement |
|---|---|
| `NavController` | Custom navigator class backed by a `MutableBackStack<NavKey>` |
| `NavGraphBuilder` + `composable<>` | `EntryProviderScope<NavKey>` + `entry<>` |
| `NavHost` | `NavDisplay` |
| `Bundle` arguments | `@Serializable data class` implementing `NavKey` |
| `savedStateHandle` result | `ResultStore` + `CompositionLocal` |

**Core rule: ALL navigation logic lives in the `app` module.** Feature modules expose `Screen` composables only — they have no dependency on Nav3 types.

---

## Directory Structure

```
app/src/main/java/<package>/navigation/
├── root/
│   ├── RootNavigation.kt        # Root NavDisplay — full-screen flows
│   └── RootNavigator.kt         # navigate(), popBackStack(), replaceTop()
├── bottombar/
│   ├── BottomBarNavigation.kt   # BottomBar NavDisplay — tab sub-screens
│   └── BottomBarNavigator.kt    # goBack(), navigate(), reSelectTab()
├── result/
│   └── ResultStore.kt           # ResultStore class + LocalResultStore CompositionLocal
├── utils/
│   └── NavigationTransitions.kt # Transition specs and metadata helpers
└── [feature]/
    └── [Feature]Navigation.kt   # Entry function + NavKey route (+ ResultKey if consumer)
```

---

## 1. NavKey Routes

Every destination is a `@Serializable` Kotlin type implementing `NavKey`.

```kotlin
// Object route — no parameters
@Serializable
object HomeRoute : NavKey

// Data class route — type-safe parameters, no bundles
@Serializable
data class ProductDetailRoute(val productId: String) : NavKey

// Navigate
navigator.navigate(ProductDetailRoute(productId = "abc-123"))

// Receive in entry — parameters are directly on the key
entry<ProductDetailRoute> { key ->
    ProductDetailScreen(productId = key.productId, ...)
}
```

---

## 2. Navigator

Create a thin wrapper around `MutableBackStack<NavKey>` for each NavDisplay level.

```kotlin
// RootNavigator.kt
class RootNavigator(private val backStack: MutableBackStack<NavKey>) {
    fun navigate(route: NavKey) { backStack.add(route) }
    fun popBackStack() { backStack.removeLastOrNull() }
    fun replaceTop(route: NavKey) {
        backStack.removeLastOrNull()
        backStack.add(route)
    }
}

// BottomBarNavigator.kt
class BottomBarNavigator(private val backStack: MutableBackStack<NavKey>) {
    fun navigate(route: NavKey) { backStack.add(route) }
    fun goBack() { backStack.removeLastOrNull() }
    fun reSelectTab(tab: NavKey) {
        backStack.removeAll { it == tab }
        backStack.add(tab)
    }
}
```

---

## 3. NavDisplay Setup

```kotlin
// RootNavigation.kt
@Composable
fun RootNavigation() {
    val backStack = rememberMutableBackStack<NavKey>(listOf(HomeRoute))
    val resultStore = remember { ResultStore() }
    val rootNavigator = remember { RootNavigator(backStack) }

    CompositionLocalProvider(LocalResultStore provides resultStore) {
        NavDisplay(
            backStack = backStack,
            entryProvider = entryProvider {
                homeEntry(rootNavigator)
                productDetailEntry(rootNavigator)
                // register all full-screen entries here
            }
        )
    }
}
```

---

## 4. Entry Patterns

### Basic Entry

```kotlin
fun EntryProviderScope<NavKey>.productDetailEntry(rootNavigator: RootNavigator) {
    entry<ProductDetailRoute>(metadata = horizontalSlideMetadata()) {
        val viewModel: ProductDetailViewModel = koinViewModel()
        val state by viewModel.viewState.collectAsStateWithLifecycle()

        ProductDetailScreen(
            state = state,
            onEvent = viewModel::onViewEvent,
            sideEffects = viewModel.sideEffect,
            onNavigation = { effect ->
                when (effect) {
                    ProductDetailSideEffect.Navigation.Back ->
                        rootNavigator.popBackStack()
                    is ProductDetailSideEffect.Navigation.OpenRelated ->
                        rootNavigator.navigate(ProductDetailRoute(effect.productId))
                }
            },
        )
    }
}

@Serializable
data class ProductDetailRoute(val productId: String) : NavKey
```

### Tab Root Screen (no animation)

```kotlin
entry<BottomTab.Home>(metadata = noAnimationMetadata()) {
    val viewModel: HomeViewModel = koinViewModel()
    val state by viewModel.viewState.collectAsStateWithLifecycle()
    HomeScreen(state = state, onEvent = viewModel::onViewEvent, ...)
}
```

### BottomBar Sub-Screen

```kotlin
fun EntryProviderScope<NavKey>.settingsEntry(navigator: BottomBarNavigator) {
    entry<SettingsRoute>(metadata = horizontalSlideMetadata()) {
        val viewModel: SettingsViewModel = koinViewModel()
        val state by viewModel.viewState.collectAsStateWithLifecycle()

        SettingsScreen(
            state = state,
            onEvent = viewModel::onViewEvent,
            sideEffects = viewModel.sideEffect,
            onNavigation = { effect ->
                when (effect) {
                    SettingsSideEffect.Navigation.Back -> navigator.goBack()
                }
            },
        )
    }
}

@Serializable
object SettingsRoute : NavKey
```

### Sub-Screen That Also Launches Full-Screen Flow

```kotlin
fun EntryProviderScope<NavKey>.profileEntry(
    rootNavigator: RootNavigator,
    navigator: BottomBarNavigator,
) {
    entry<ProfileRoute>(metadata = horizontalSlideMetadata()) {
        // ...
        onNavigation = { effect ->
            when (effect) {
                ProfileSideEffect.Navigation.Back -> navigator.goBack()
                ProfileSideEffect.Navigation.OpenSetup -> rootNavigator.navigate(SetupRoute)
            }
        }
    }
}
```

---

## 5. Transition Helpers

```kotlin
// NavigationTransitions.kt
fun horizontalSlideMetadata(): NavEntryMetadata = /* slide in from end, slide out to end */
fun noAnimationMetadata(): NavEntryMetadata = /* no transition, instant switch */
fun verticalSlideMetadata(): NavEntryMetadata = /* slide up from bottom */
```

Use `noAnimationMetadata()` for tab roots, `horizontalSlideMetadata()` for drill-down screens, and `verticalSlideMetadata()` for full-screen modal flows (e.g., onboarding, setup wizards).

---

## 6. ResultStore — Cross-Screen Results

`ResultStore` passes data **back** from a producer screen to its consumer without coupling them via parameters.

### Signal (no data)

```kotlin
// --- Consumer (CheckoutNavigation.kt) ---
fun EntryProviderScope<NavKey>.checkoutEntry(rootNavigator: RootNavigator) {
    entry<CheckoutRoute> {
        val resultStore = LocalResultStore.current
        val paymentDone = resultStore.getResult<PaymentCompleted>(PaymentCompletedKey)

        LaunchedEffect(paymentDone) {
            if (paymentDone != null) {
                resultStore.removeResult(PaymentCompletedKey)
                rootNavigator.popBackStack()
            }
        }
        // ...
    }
}

data object PaymentCompleted
object PaymentCompletedKey

// --- Producer (PaymentNavigation.kt) ---
onNavigation = { effect ->
    when (effect) {
        PaymentSideEffect.Navigation.Success -> {
            resultStore.setResult(PaymentCompletedKey, PaymentCompleted)
            rootNavigator.popBackStack()
        }
    }
}
```

### Data Payload

```kotlin
// Consumer (OrderNavigation.kt)
val result = resultStore.getResult<AddressResult>(AddressResultKey)
LaunchedEffect(result) {
    result?.let {
        viewModel.onViewEvent(OrderViewEvent.AddressSelected(it.addressId))
        resultStore.removeResult(AddressResultKey)
    }
}

data class AddressResult(val addressId: String)
object AddressResultKey
```

### Multiple Result Types (sealed interface)

```kotlin
// Consumer (FeedNavigation.kt)
val scrollRequest = resultStore.getResult<FeedScrollRequest>(FeedScrollRequestKey)
LaunchedEffect(scrollRequest) {
    if (scrollRequest != null) {
        resultStore.removeResult(FeedScrollRequestKey)
        when (scrollRequest) {
            is FeedScrollRequest.ToItem ->
                viewModel.onViewEvent(FeedViewEvent.ScrollToItem(scrollRequest.itemId))
            FeedScrollRequest.ResetToTop ->
                viewModel.onViewEvent(FeedViewEvent.ResetToTop)
        }
    }
}

sealed interface FeedScrollRequest {
    data class ToItem(val itemId: String) : FeedScrollRequest
    data object ResetToTop : FeedScrollRequest
}
object FeedScrollRequestKey
```

---

## ResultStore Implementation

```kotlin
// ResultStore.kt
class ResultStore {
    private val results = mutableStateMapOf<Any, Any?>()

    fun <T : Any> setResult(key: Any, value: T) { results[key] = value }
    fun <T : Any> getResult(key: Any): T? = results[key] as? T
    fun removeResult(key: Any) { results.remove(key) }
}

val LocalResultStore = staticCompositionLocalOf<ResultStore> {
    error("No ResultStore provided")
}
```

---

## Do / Don't

| ✅ Do | ❌ Don't |
|---|---|
| Put navigation wiring in `app/navigation/[feature]/` | Put navigation code in `feature/`, `component/`, or `common/` modules |
| Use `@Serializable data class/object` routes implementing `NavKey` | Use string routes or `Bundle` arguments |
| Access `ResultStore` via `LocalResultStore.current` inside entry lambda | Pass `ResultStore` as a function parameter |
| Use `koinViewModel()` (or equivalent DI) in entries | Instantiate ViewModels manually |
| Emit `Navigation.*` side effects from ViewModels | Call navigator methods from composables directly |
| Use `noAnimationMetadata()` for tab roots | Animate tab root switches |
| Use `horizontalSlideMetadata()` for drill-down screens | Mix transition styles inconsistently |
| Handle back via `Navigation.Back` side effect | Use `BackHandler` in screen composables |
| Use `remember { ResultStore() }` — plain remember | Use `rememberSaveable { ResultStore() }` — crashes, not Bundle-serializable |
| Register each route in exactly one `NavDisplay` | Register the same route in multiple NavDisplays |

---

## Verification Checklist

- [ ] Route is `@Serializable` and implements `NavKey`
- [ ] Entry registered in exactly one NavDisplay
- [ ] ViewModel resolved with `koinViewModel()` (or your DI)
- [ ] No `BackHandler` in the Screen composable
- [ ] Correct animation metadata applied (`noAnimationMetadata` vs `horizontalSlideMetadata`)
- [ ] `ResultStore` accessed via `LocalResultStore.current` (never as parameter)
- [ ] `ResultKey` + payload type defined in the **consumer** file
- [ ] Producer imports `ResultKey` from the consumer file
- [ ] No navigation logic in any module other than `app`
- [ ] `ResultStore` created with `remember { }`, not `rememberSaveable { }`

---

## References

- [Jetpack Navigation 3 — official docs](https://developer.android.com/guide/navigation/navigation3)
- [Navigation 3 — Getting Started codelab](https://developer.android.com/codelabs/navigation3)
- [kotlinx.serialization](https://github.com/Kotlin/kotlinx.serialization)
- [Compose side effects (LaunchedEffect)](https://developer.android.com/develop/ui/compose/side-effects)
