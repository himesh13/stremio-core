# Project Structure and File Organization

This document outlines the recommended project structure for the Android app using stremio-core.

## Repository Structure

```
stremio-core/
├── src/                                    # Core Rust library
│   ├── models/                             # State models
│   ├── types/                              # Data types
│   ├── runtime/                            # Runtime and Env
│   └── addon_transport/                    # Addon communication
│
├── stremio-android-bridge/                 # New Android bridge (to be created)
│   ├── src/
│   │   ├── lib.rs                         # Main entry point
│   │   ├── env.rs                         # AndroidEnv implementation
│   │   ├── runtime_bridge.rs              # Runtime wrapper
│   │   ├── models/                        # Model bridges
│   │   │   ├── ctx_bridge.rs
│   │   │   ├── library_bridge.rs
│   │   │   ├── player_bridge.rs
│   │   │   └── ...
│   │   └── stremio_android.udl           # uniffi interface definition
│   ├── Cargo.toml                         # Rust dependencies
│   ├── build.rs                           # Build script for uniffi
│   └── README.md                          # Bridge documentation
│
├── android/                                # Android app (to be created)
│   ├── app/
│   │   ├── src/
│   │   │   ├── main/
│   │   │   │   ├── java/com/stremio/android/
│   │   │   │   │   ├── MainActivity.kt
│   │   │   │   │   ├── StremioApp.kt    # Application class
│   │   │   │   │   ├── di/               # Hilt modules
│   │   │   │   │   │   ├── AppModule.kt
│   │   │   │   │   │   ├── CoreModule.kt
│   │   │   │   │   │   └── ...
│   │   │   │   │   ├── data/             # Data layer
│   │   │   │   │   │   ├── repository/
│   │   │   │   │   │   │   ├── StremioRepository.kt
│   │   │   │   │   │   │   └── ...
│   │   │   │   │   │   └── models/       # Kotlin data classes
│   │   │   │   │   ├── domain/           # Domain layer
│   │   │   │   │   │   └── usecases/
│   │   │   │   │   ├── ui/               # UI layer
│   │   │   │   │   │   ├── theme/        # Compose theme
│   │   │   │   │   │   ├── components/   # Reusable components
│   │   │   │   │   │   ├── screens/      # Screen composables
│   │   │   │   │   │   │   ├── home/
│   │   │   │   │   │   │   │   ├── HomeScreen.kt
│   │   │   │   │   │   │   │   └── HomeViewModel.kt
│   │   │   │   │   │   │   ├── library/
│   │   │   │   │   │   │   ├── details/
│   │   │   │   │   │   │   ├── player/
│   │   │   │   │   │   │   └── ...
│   │   │   │   │   │   └── navigation/   # Navigation setup
│   │   │   │   │   └── core/             # Core Android utils
│   │   │   │   │       └── bridge/       # Rust bridge wrapper
│   │   │   │   │           └── CoreBridge.kt
│   │   │   │   ├── jniLibs/              # Native libraries
│   │   │   │   │   ├── arm64-v8a/
│   │   │   │   │   │   └── libstremio_android_bridge.so
│   │   │   │   │   ├── armeabi-v7a/
│   │   │   │   │   │   └── libstremio_android_bridge.so
│   │   │   │   │   ├── x86/
│   │   │   │   │   │   └── libstremio_android_bridge.so
│   │   │   │   │   └── x86_64/
│   │   │   │   │       └── libstremio_android_bridge.so
│   │   │   │   ├── res/                  # Android resources
│   │   │   │   └── AndroidManifest.xml
│   │   │   ├── test/                     # Unit tests
│   │   │   └── androidTest/              # Integration tests
│   │   ├── build.gradle.kts              # App build config
│   │   └── proguard-rules.pro            # ProGuard rules
│   ├── build.gradle.kts                  # Project build config
│   ├── settings.gradle.kts               # Project settings
│   ├── gradle.properties
│   └── local.properties                  # Local SDK paths
│
├── docs/                                   # Documentation
│   └── android/                           # Android-specific docs
│       ├── README.md
│       ├── ANDROID_APP_PLAN.md
│       ├── QUICK_START.md
│       ├── ENVIRONMENT_IMPLEMENTATION.md
│       ├── ARCHITECTURE_DECISIONS.md
│       └── PROJECT_STRUCTURE.md          # This file
│
├── Cargo.toml                             # Workspace config
├── Cargo.lock
└── README.md
```

## Detailed Component Breakdown

### stremio-android-bridge (Rust)

#### lib.rs
Main entry point that exports functions via uniffi.

```rust
// Example structure
mod env;
mod runtime_bridge;
mod models;
mod error;

pub use env::AndroidEnv;
pub use runtime_bridge::RuntimeBridge;
pub use error::CoreError;

uniffi::include_scaffolding!("stremio_android");
```

#### env.rs
AndroidEnv implementation of the Env trait.

Key responsibilities:
- HTTP networking
- Local storage
- Addon transport
- Analytics

#### runtime_bridge.rs
Wrapper around stremio-core Runtime for Android.

Key functions:
- `initialize_runtime()`
- `dispatch_action()`
- `observe_state()`
- `get_state()`

#### models/
Individual bridge modules for each model:
- ctx_bridge.rs - User context and authentication
- library_bridge.rs - Library management
- player_bridge.rs - Player state
- catalog_bridge.rs - Catalog browsing
- etc.

#### stremio_android.udl
uniffi interface definition file.

```udl
namespace stremio_android {
    RuntimeBridge initialize_runtime(string storage_path);
    // ... more functions
};

interface RuntimeBridge {
    void dispatch_action(CoreAction action);
    CoreState get_state();
    // ... more methods
};

// ... more definitions
```

### Android App (Kotlin)

#### app/src/main/java/com/stremio/android/

##### di/ (Dependency Injection)
Hilt modules for dependency injection.

**AppModule.kt**:
```kotlin
@Module
@InstallIn(SingletonComponent::class)
object AppModule {
    @Provides
    @Singleton
    fun provideCoreBridge(
        @ApplicationContext context: Context
    ): CoreBridge {
        val storagePath = context.filesDir.absolutePath + "/stremio"
        return CoreBridge.initialize(storagePath)
    }
}
```

##### data/repository/
Repository pattern implementation.

**StremioRepository.kt**:
```kotlin
interface StremioRepository {
    fun observeLibrary(): Flow<Result<Library>>
    suspend fun addToLibrary(metaId: String): Result<Unit>
    // ... more methods
}
```

##### ui/screens/
Screen-specific composables and ViewModels.

Directory structure:
```
screens/
├── home/
│   ├── HomeScreen.kt
│   ├── HomeViewModel.kt
│   └── components/
├── library/
│   ├── LibraryScreen.kt
│   ├── LibraryViewModel.kt
│   └── components/
├── details/
│   ├── DetailsScreen.kt
│   ├── DetailsViewModel.kt
│   └── components/
└── player/
    ├── PlayerScreen.kt
    ├── PlayerViewModel.kt
    └── components/
```

##### ui/navigation/
Navigation setup using Jetpack Navigation Compose.

**NavGraph.kt**:
```kotlin
@Composable
fun StremioNavGraph(
    navController: NavHostController,
    startDestination: String = Screen.Home.route
) {
    NavHost(
        navController = navController,
        startDestination = startDestination
    ) {
        composable(Screen.Home.route) {
            HomeScreen(navController)
        }
        composable(Screen.Library.route) {
            LibraryScreen(navController)
        }
        // ... more screens
    }
}
```

##### core/bridge/
Wrapper around uniffi-generated code.

**CoreBridge.kt**:
```kotlin
class CoreBridge private constructor(
    private val runtime: RuntimeBridge
) {
    companion object {
        fun initialize(storagePath: String): CoreBridge {
            val runtime = initializeRuntime(storagePath)
            return CoreBridge(runtime)
        }
    }
    
    fun dispatchAction(action: CoreAction) {
        runtime.dispatchAction(action)
    }
    
    fun observeLibrary(callback: (LibraryState) -> Unit) {
        // Setup observer
    }
}
```

## File Naming Conventions

### Rust
- Snake case for files: `runtime_bridge.rs`
- PascalCase for types: `struct AndroidEnv`
- snake_case for functions: `fn initialize_runtime()`

### Kotlin
- PascalCase for files: `HomeScreen.kt`
- PascalCase for classes: `class HomeViewModel`
- camelCase for functions: `fun dispatchAction()`
- camelCase for variables: `val storagePath`

## Build Configuration Files

### Cargo.toml (Root)
```toml
[workspace]
members = [
    "stremio-core-web",
    "stremio-derive",
    "stremio-watched-bitfield",
    "stremio-android-bridge"
]
resolver = "2"
```

### stremio-android-bridge/Cargo.toml
```toml
[package]
name = "stremio-android-bridge"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib"]
name = "stremio_android_bridge"

[dependencies]
stremio-core = { path = "../" }
uniffi = "0.25"
# ... more dependencies

[build-dependencies]
uniffi = { version = "0.25", features = ["build"] }
```

### android/build.gradle.kts (Root)
```kotlin
plugins {
    id("com.android.application") version "8.2.0" apply false
    id("org.jetbrains.kotlin.android") version "1.9.22" apply false
    id("com.google.dagger.hilt.android") version "2.50" apply false
}
```

### android/app/build.gradle.kts
```kotlin
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
    id("kotlin-kapt")
    id("com.google.dagger.hilt.android")
}

android {
    namespace = "com.stremio.android"
    compileSdk = 34
    
    defaultConfig {
        applicationId = "com.stremio.android"
        minSdk = 24
        targetSdk = 34
        
        ndk {
            abiFilters.addAll(listOf("arm64-v8a", "armeabi-v7a"))
        }
    }
    
    // ... more configuration
}

// Task to build Rust library
tasks.register<Exec>("buildRustLib") {
    workingDir = file("../../stremio-android-bridge")
    commandLine = listOf(
        "cargo", "ndk",
        "-t", "armeabi-v7a",
        "-t", "arm64-v8a",
        "-o", "../android/app/src/main/jniLibs",
        "build", "--release"
    )
}

// Make buildRustLib run before compilation
tasks.whenTaskAdded {
    if (name == "preBuild") {
        dependsOn("buildRustLib")
    }
}
```

## Git Configuration

### .gitignore Additions
```gitignore
# Android
android/local.properties
android/.gradle/
android/build/
android/app/build/
android/.idea/
android/*.iml

# Rust
stremio-android-bridge/target/
stremio-android-bridge/Cargo.lock

# Native libraries (committed)
!android/app/src/main/jniLibs/**/*.so

# Generated bindings
android/app/src/main/java/uniffi/

# Build artifacts
*.apk
*.aab
```

## Script Organization

### build_scripts/
Create a directory for build automation:

```
build_scripts/
├── build_android.sh           # Build entire Android app
├── build_rust.sh             # Build Rust library only
├── generate_bindings.sh      # Generate Kotlin bindings
├── clean.sh                  # Clean all build artifacts
└── release.sh                # Build release APK/AAB
```

Example `build_android.sh`:
```bash
#!/bin/bash
set -e

echo "Building Rust library..."
cd stremio-android-bridge
cargo ndk \
    -t armeabi-v7a \
    -t arm64-v8a \
    -o ../android/app/src/main/jniLibs \
    build --release

echo "Generating Kotlin bindings..."
uniffi-bindgen generate \
    src/stremio_android.udl \
    --language kotlin \
    --out-dir ../android/app/src/main/java

echo "Building Android app..."
cd ../android
./gradlew assembleDebug

echo "Build complete!"
```

## Testing Structure

### Rust Tests
```
stremio-android-bridge/
├── src/
│   └── lib.rs
└── tests/
    ├── env_tests.rs
    ├── runtime_tests.rs
    └── integration_tests.rs
```

### Kotlin Tests
```
app/src/
├── test/                      # Unit tests
│   └── com/stremio/android/
│       ├── repository/
│       ├── viewmodel/
│       └── bridge/
└── androidTest/               # Integration tests
    └── com/stremio/android/
        ├── ui/
        └── integration/
```

## Documentation Structure

```
docs/
├── android/
│   ├── README.md              # Navigation guide
│   ├── ANDROID_APP_PLAN.md   # Complete plan
│   ├── QUICK_START.md        # Quick start guide
│   ├── ENVIRONMENT_IMPLEMENTATION.md
│   ├── ARCHITECTURE_DECISIONS.md
│   ├── PROJECT_STRUCTURE.md  # This file
│   ├── API_REFERENCE.md      # API documentation
│   └── TROUBLESHOOTING.md    # Common issues
└── diagrams/                 # Architecture diagrams
    ├── architecture.png
    ├── data_flow.png
    └── component_diagram.png
```

## Development Workflow

1. **Make changes to Rust code** in `stremio-android-bridge/`
2. **Build Rust library**: `./build_scripts/build_rust.sh`
3. **Generate bindings**: `./build_scripts/generate_bindings.sh`
4. **Make changes to Kotlin code** in `android/app/src/`
5. **Build and test**: `cd android && ./gradlew assembleDebug test`
6. **Run on device**: Android Studio or `./gradlew installDebug`

## Recommended IDE Setup

### Rust Development
- **IDE**: VS Code or IntelliJ IDEA with Rust plugin
- **Extensions**: rust-analyzer, CodeLLDB

### Android Development
- **IDE**: Android Studio
- **Plugins**: Kotlin, Jetpack Compose Preview

### Unified Development
- **Option 1**: Use IntelliJ IDEA Ultimate (supports both)
- **Option 2**: Use both IDEs, with Android Studio for UI

## Next Steps

1. Create the directory structure as outlined above
2. Follow [QUICK_START.md](./QUICK_START.md) to set up the bridge
3. Implement core components following [ANDROID_APP_PLAN.md](./ANDROID_APP_PLAN.md)
4. Build features incrementally, testing as you go

## References

- [Android App Architecture Guide](https://developer.android.com/topic/architecture)
- [Jetpack Compose Guidelines](https://developer.android.com/jetpack/compose/designsystems/anatomy)
- [Rust Project Structure](https://doc.rust-lang.org/cargo/guide/project-layout.html)
- [uniffi-rs Best Practices](https://mozilla.github.io/uniffi-rs/latest/)
