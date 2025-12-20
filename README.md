# FeatureFlow

[![Platform](https://img.shields.io/badge/Platform-Android-green.svg)](https://developer.android.com)
[![API](https://img.shields.io/badge/API-31%2B-brightgreen.svg)](https://android-arsenal.com/api?level=31)
[![Kotlin](https://img.shields.io/badge/Kotlin-1.9.22-blue.svg)](https://kotlinlang.org)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)

A robust Android library for managing dynamic feature modules with on-demand delivery. FeatureFlow leverages Google Play Core's SplitInstall API to enable seamless feature loading, installation, and display while reducing initial app size.

## Features

- **On-Demand Feature Loading** - Download and install feature modules only when needed
- **Real-Time Progress Tracking** - Monitor installation progress with user confirmation support
- **URI-Based Routing** - Deep link support with automatic feature resolution
- **Interceptor System** - Pre and post-installation hooks for custom logic
- **State Persistence** - Automatic recovery from interrupted installations
- **Jetpack Compose UI** - Modern reactive UI with Material Design
- **Coroutine-Based Architecture** - Fully async/reactive with Flow patterns
- **Comprehensive Error Handling** - Detailed error mapping with recovery mechanisms

## Requirements

- **Minimum SDK**: API 31 (Android 12)
- **Target SDK**: API 34 (Android 14)
- **Java**: 17
- **Kotlin**: 1.9.22

## Installation

Add the dependency to your module's `build.gradle.kts`:

```kotlin
dependencies {
    implementation("com.kuru:featureflow:7.0.1")
}
```

Or if using Gradle Groovy:

```groovy
dependencies {
    implementation 'com.kuru:featureflow:7.0.1'
}
```

## Quick Start

### 1. Register Your Feature

```kotlin
@HiltViewModel
class MainViewModel @Inject constructor(
    private val featureRegistry: DFFeatureRegistryUseCase
) : ViewModel() {

    init {
        featureRegistry.registerFeature(
            DFFeatureConfig(
                featureName = "premium_feature",
                interceptors = listOf(
                    DFFeatureInterceptor(
                        name = "auth_check",
                        preInstall = true,
                        task = { checkUserAuthentication() }
                    )
                )
            )
        )
    }
}
```

### 2. Implement Feature Provider

In your dynamic feature module, implement `DFFeatureProvider`:

```kotlin
class PremiumFeatureProvider : DFFeatureProvider {
    override fun getScreen(): @Composable () -> Unit = {
        PremiumFeatureScreen()
    }
}
```

### 3. Configure ServiceLoader

Create a file at `src/main/resources/META-INF/services/com.kuru.featureflow.component.register.DFFeatureProvider`:

```
com.yourapp.premium.PremiumFeatureProvider
```

### 4. Load Features via URI

```kotlin
// Deep link or programmatic navigation
val intent = Intent(Intent.ACTION_VIEW).apply {
    data = Uri.parse("app://feature/premium_feature")
    setClass(context, DFComponentActivity::class.java)
}
startActivity(intent)
```

## Architecture

FeatureFlow follows a clean, layered architecture with clear separation of concerns:

```
┌─────────────────────────────────────────────────────────────┐
│                        UI Layer                              │
│  ┌─────────────────┐  ┌──────────────────┐  ┌────────────┐  │
│  │ DFComponent     │  │ DFComponent      │  │ DFComponent│  │
│  │ Activity        │──│ ViewModel        │──│ Screen     │  │
│  └─────────────────┘  └──────────────────┘  └────────────┘  │
└─────────────────────────────┬───────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────┐
│                      Domain Layer                            │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐ │
│  │ ResolveFeature │  │ LoadFeature    │  │ InstallFeature │ │
│  │ RouteUseCase   │  │ UseCase        │  │ UseCase        │ │
│  └────────────────┘  └────────────────┘  └────────────────┘ │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐ │
│  │ TrackFeature   │  │ HandleFeature  │  │ CompleteSetup  │ │
│  │ InstallUseCase │  │ Interceptors   │  │ UseCase        │ │
│  └────────────────┘  └────────────────┘  └────────────────┘ │
└─────────────────────────────┬───────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────┐
│                 State Management Layer                       │
│  ┌─────────────────────────────────────────────────────────┐│
│  │                    DFStateStore                          ││
│  │  • Persistent State (DataStore)                          ││
│  │  • In-Memory State (StateFlow)                           ││
│  └─────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
```

### Layers

| Layer | Responsibility |
|-------|----------------|
| **UI** | Activity, ViewModel, and Composable screens for user interaction |
| **Domain** | Use cases implementing core business logic |
| **State** | State management with DataStore and StateFlow |
| **Registration** | Feature configuration and interceptor management |
| **DI** | Hilt-based dependency injection |

## Feature Loading Flow

```
1. URI Request → DFComponentActivity receives Intent
                        │
2. URI Parsing → DFResolveFeatureRouteUseCase extracts feature info
                        │
3. Load Check → DFLoadFeatureUseCase verifies registration & status
                        │
4. Pre-Install → DFHandleFeatureInterceptorsUseCase runs hooks
                        │
5. Installation → DFInstallFeatureUseCase downloads via SplitInstall
                        │
6. Monitoring → DFTrackFeatureInstallUseCase tracks progress
                        │
7. Post-Install → DFCompleteFeatureSetupUseCase initializes feature
                        │
8. Display → Feature's Composable screen is rendered
```

## Configuration

### Feature Configuration

```kotlin
DFFeatureConfig(
    featureName = "my_feature",
    interceptors = listOf(
        // Pre-install: runs before download
        DFFeatureInterceptor(
            name = "pre_check",
            preInstall = true,
            task = { performPreCheck() }
        ),
        // Post-install: runs after installation
        DFFeatureInterceptor(
            name = "post_setup",
            preInstall = false,
            task = { performPostSetup() }
        )
    )
)
```

### State Observation

```kotlin
@Composable
fun FeatureLoadingScreen(viewModel: DFComponentViewModel) {
    val uiState by viewModel.uiState.collectAsState()

    when (uiState) {
        is DFComponentState.Loading -> LoadingIndicator()
        is DFComponentState.Error -> ErrorScreen(
            message = (uiState as DFComponentState.Error).message,
            onRetry = { viewModel.processIntent(DFComponentIntent.Retry) }
        )
        is DFComponentState.RequiresConfirmation -> ConfirmationDialog()
        is DFComponentState.Success -> { /* Feature loaded */ }
    }
}
```

## Building

```bash
# Clean and build
./gradlew clean build

# Run unit tests
./gradlew test

# Run instrumentation tests
./gradlew connectedAndroidTest

# Build release AAR
./gradlew assembleRelease

# Publish to local Maven repository
./gradlew publishToMavenLocal
```

## Testing

The library includes comprehensive test coverage:

- **Unit Tests**: MockK and Mockito-based testing for all use cases
- **ViewModel Tests**: State management and intent handling verification
- **Robolectric Tests**: Android context testing without emulator
- **Integration Tests**: End-to-end feature loading scenarios

Run tests with:

```bash
./gradlew test              # Unit tests
./gradlew connectedCheck    # Instrumentation tests
```

## Dependencies

| Library | Purpose |
|---------|---------|
| Jetpack Compose | Modern declarative UI |
| Hilt | Dependency injection |
| Play Core | SplitInstall for dynamic delivery |
| DataStore | Persistent state storage |
| Kotlin Coroutines | Async operations with Flow |
| Navigation Compose | In-app navigation |

## License

```
Copyright 2024 Kuru

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## Documentation

For detailed architecture documentation, see [ReadMe](ReadMe).
