# Comprehensive Plan for Building a Stremio Android App

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Repository Overview](#repository-overview)
3. [Architecture Strategy](#architecture-strategy)
4. [Development Phases](#development-phases)
5. [Technical Stack](#technical-stack)
6. [Implementation Roadmap](#implementation-roadmap)
7. [Key Components](#key-components)
8. [Integration Patterns](#integration-patterns)
9. [Testing Strategy](#testing-strategy)
10. [Deployment & Distribution](#deployment--distribution)

---

## Executive Summary

This document outlines a comprehensive plan for building an Android application that mimics the Stremio Android app functionality using the `stremio-core` Rust library. The `stremio-core` repository contains the core business logic for the Stremio media center application, written in Rust, and is designed to be reusable across different platforms.

### Key Goals

- **Reuse Core Logic**: Leverage the existing Rust codebase for all business logic
- **Native Android UI**: Build a modern Android UI using Kotlin/Jetpack Compose
- **Platform Integration**: Properly integrate Rust core with Android via JNI/FFI
- **Feature Parity**: Match the functionality of the official Stremio Android app
- **Performance**: Ensure smooth, performant user experience
- **Maintainability**: Create a clean, well-documented architecture

---

## Repository Overview

### What is stremio-core?

`stremio-core` is a Rust crate containing all the reusable logic for Stremio:

- **Purpose**: Full-featured media center for organizing and streaming videos, movies, and TV series
- **Language**: Rust (edition 2021, MSRV 1.77)
- **Architecture**: Inspired by Elm architecture with effects and update patterns
- **Flexibility**: Designed to integrate across the entire stack and different paradigms

### Key Modules in stremio-core

1. **types**: Core data types and structures
2. **addon_transport**: Communication with add-ons, legacy protocol adapter
3. **state_types**: Application state types with Effects and Update traits
4. **runtime**: Handles effects automatically for applications
5. **environment**: Trait describing the environment (fetch, storage)
6. **models**: Stateful models including:
   - `Context` (user authentication, add-ons)
   - `Library`
   - `CatalogFiltered`
   - `MetaDetails`
   - `Player`
   - `StreamingServer`
   - And more...

### Existing Platform Implementations

#### stremio-core-web (WASM)

Located in `stremio-core-web/`, this serves as a **reference implementation** for platform integration:

- Uses WebAssembly (WASM) to bridge Rust core with JavaScript
- Implements `WebEnv` environment trait
- Exports WASM bindings via `wasm-bindgen`
- Provides serialization models for web
- Shows patterns for:
  - Environment implementation
  - Model serialization
  - Event handling
  - Runtime management

**Key Lesson**: The web implementation demonstrates how to wrap the core library for a specific platform.

---

## Architecture Strategy

### Recommended Approach: JNI Bridge with uniffi

We recommend using **Mozilla's uniffi-rs** for generating JNI bindings automatically.

#### Why uniffi-rs?

1. **Automatic Binding Generation**: Generates Kotlin bindings from Rust code
2. **Type Safety**: Ensures type-safe communication between Rust and Kotlin
3. **Reduces Boilerplate**: No manual JNI code required
4. **Well-Maintained**: Used by Mozilla in Firefox and other projects
5. **Clean API**: Generates idiomatic Kotlin code

#### Alternative Approaches

| Approach | Pros | Cons | Recommendation |
|----------|------|------|----------------|
| **uniffi-rs** | Type-safe, automatic, clean | Some learning curve | ✅ **Recommended** |
| Manual JNI | Full control | Very tedious, error-prone | ❌ Avoid |
| flutter_rust_bridge | Good Flutter integration | Requires Flutter | Only if using Flutter |
| WASM + WebView | Fast development | Performance overhead, not native | ❌ Avoid for production |

### High-Level Architecture

```
┌─────────────────────────────────────────────┐
│          Android App (Kotlin)                │
│                                              │
│  ┌────────────────────────────────────────┐ │
│  │   UI Layer (Jetpack Compose)           │ │
│  │   - Screens                            │ │
│  │   - Navigation                         │ │
│  │   - UI Components                      │ │
│  └────────────────┬───────────────────────┘ │
│                   │                          │
│  ┌────────────────▼───────────────────────┐ │
│  │   ViewModel Layer                       │ │
│  │   - State Management                    │ │
│  │   - Business Logic Coordination         │ │
│  └────────────────┬───────────────────────┘ │
│                   │                          │
│  ┌────────────────▼───────────────────────┐ │
│  │   Kotlin Bindings (Generated)          │ │
│  │   - Type-safe FFI calls                │ │
│  │   - Data serialization                 │ │
│  └────────────────┬───────────────────────┘ │
│                   │ JNI/FFI                  │
└───────────────────┼──────────────────────────┘
                    │
┌───────────────────▼──────────────────────────┐
│   stremio-android-bridge (Rust)              │
│   - uniffi definitions                       │
│   - Android environment implementation       │
│   - Platform-specific adapters               │
└───────────────────┬──────────────────────────┘
                    │
┌───────────────────▼──────────────────────────┐
│   stremio-core (Rust)                        │
│   - Core business logic                      │
│   - Models (Ctx, Library, Player, etc.)      │
│   - Types and data structures                │
│   - Addon transport                          │
│   - Runtime and effects                      │
└──────────────────────────────────────────────┘
```

---

## Development Phases

### Phase 1: Foundation (Weeks 1-3)

**Goal**: Set up development environment and basic infrastructure

#### Tasks:
- [ ] Set up Rust development environment
- [ ] Set up Android Studio and Kotlin development
- [ ] Install uniffi-rs and dependencies
- [ ] Create new Android project structure
- [ ] Set up Gradle build configuration for Rust
- [ ] Create basic uniffi bridge crate
- [ ] Implement hello-world FFI call
- [ ] Set up CI/CD pipeline basics
- [ ] Configure logging and debugging tools

#### Deliverables:
- Working Android project that can call Rust code
- Build scripts that compile Rust for Android targets
- Basic project documentation

### Phase 2: Environment Implementation (Weeks 4-6)

**Goal**: Implement Android-specific environment trait

#### Tasks:
- [ ] Study `stremio-core-web/src/env.rs` as reference
- [ ] Implement `Env` trait for Android
  - [ ] HTTP fetch using Android's networking stack
  - [ ] Storage using SharedPreferences/Room
  - [ ] Addon transport setup
  - [ ] Analytics integration (optional)
- [ ] Implement proper error handling
- [ ] Add logging integration
- [ ] Write unit tests for environment
- [ ] Handle Android-specific permissions
- [ ] Implement background task handling

#### Deliverables:
- Complete `AndroidEnv` implementation
- Storage layer working with Android
- Network layer properly configured
- Test suite for environment

### Phase 3: Core Integration (Weeks 7-9)

**Goal**: Integrate stremio-core runtime and models

#### Tasks:
- [ ] Set up Runtime initialization
- [ ] Implement model serialization for Kotlin
- [ ] Create state management bridge
- [ ] Implement action dispatching
- [ ] Handle runtime events
- [ ] Set up observers for state changes
- [ ] Implement proper lifecycle management
- [ ] Add memory management and cleanup
- [ ] Create debugging/inspection tools

#### Deliverables:
- Runtime initialization working on Android
- State synchronization between Rust and Kotlin
- Event flow working end-to-end
- Memory-safe implementation

### Phase 4: Feature Implementation - Core Models (Weeks 10-14)

**Goal**: Implement core features using stremio-core models

#### Models to Integrate:

1. **Context (Auth & Addons)** - Week 10
   - [ ] User authentication
   - [ ] Addon management
   - [ ] Profile storage
   
2. **Library** - Week 11
   - [ ] Library items management
   - [ ] Sync functionality
   - [ ] Continue watching
   
3. **Catalog & Discovery** - Week 12
   - [ ] Catalog browsing
   - [ ] Filtering and sorting
   - [ ] Search functionality
   
4. **Meta Details** - Week 13
   - [ ] Content details view
   - [ ] Video information
   - [ ] Related content
   
5. **Player** - Week 14
   - [ ] Stream selection
   - [ ] Playback state
   - [ ] Player integration

#### Deliverables:
- All core models working on Android
- Feature parity with major functionality
- Integration tests for each model

### Phase 5: UI Development (Weeks 15-20)

**Goal**: Build Android UI with Jetpack Compose

#### Screens to Build:

1. **Authentication Flow** - Week 15
   - Login/Register screens
   - Guest mode
   - Profile management

2. **Home/Discover** - Week 16
   - Catalog display
   - Content carousels
   - Filtering UI

3. **Library** - Week 17
   - Library grid/list view
   - Continue watching
   - Sorting and filtering

4. **Details Page** - Week 18
   - Meta information display
   - Stream sources
   - Actions (add to library, etc.)

5. **Player Screen** - Week 19
   - Video player integration (ExoPlayer)
   - Playback controls
   - Subtitle management

6. **Settings & Misc** - Week 20
   - Settings screen
   - Addon management UI
   - About/Help

#### Deliverables:
- Complete UI implementation
- Navigation flow
- Responsive design
- Dark/Light theme support

### Phase 6: Platform Features (Weeks 21-23)

**Goal**: Android-specific features and optimizations

#### Tasks:
- [ ] Deep linking support
- [ ] Push notifications
- [ ] Background playback (if applicable)
- [ ] Android TV support (optional)
- [ ] Chromecast integration (optional)
- [ ] Picture-in-Picture mode
- [ ] Download management
- [ ] Offline support
- [ ] Android Auto support (optional)
- [ ] Widgets (optional)

#### Deliverables:
- Platform features working
- Enhanced user experience
- Extended functionality

### Phase 7: Polish & Optimization (Weeks 24-26)

**Goal**: Performance, testing, and quality assurance

#### Tasks:
- [ ] Performance profiling and optimization
- [ ] Memory leak detection and fixes
- [ ] Battery usage optimization
- [ ] Network efficiency
- [ ] UI/UX polish
- [ ] Accessibility improvements
- [ ] Comprehensive testing
- [ ] Bug fixing
- [ ] Documentation completion
- [ ] Code review and refactoring

#### Deliverables:
- Polished, production-ready app
- Complete test coverage
- Performance benchmarks
- User documentation

### Phase 8: Release Preparation (Weeks 27-28)

**Goal**: Prepare for production release

#### Tasks:
- [ ] Play Store listing preparation
- [ ] App screenshots and videos
- [ ] Privacy policy
- [ ] Terms of service
- [ ] Release notes
- [ ] Beta testing program
- [ ] Crash reporting setup (Firebase Crashlytics)
- [ ] Analytics setup
- [ ] ProGuard/R8 configuration
- [ ] Code signing and security
- [ ] Final QA pass

#### Deliverables:
- App ready for Play Store submission
- Marketing materials
- Support infrastructure

---

## Technical Stack

### Rust Components

#### Core Dependencies
```toml
[dependencies]
stremio-core = { path = "../stremio-core" }
uniffi = "0.25"  # or latest version

# Serialization
serde = { version = "1", features = ["derive"] }
serde_json = "1.0"

# Async runtime (Android-compatible)
tokio = { version = "1", features = ["rt", "macros", "sync"] }

# Logging
tracing = "0.1"
tracing-android = "0.1"

# HTTP client
reqwest = { version = "0.11", features = ["json"] }

# Error handling
anyhow = "1.0"
thiserror = "1.0"
```

#### Build Configuration

**Cargo.toml for Android Bridge**:
```toml
[package]
name = "stremio-android-bridge"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "staticlib"]

[[bin]]
name = "uniffi-bindgen"
path = "uniffi-bindgen.rs"

[build-dependencies]
uniffi = { version = "0.25", features = ["build"] }
```

### Android Components

#### Technology Stack
- **Language**: Kotlin (100%)
- **UI Framework**: Jetpack Compose
- **Architecture**: MVVM (Model-View-ViewModel)
- **Dependency Injection**: Hilt
- **Async**: Kotlin Coroutines + Flow
- **Navigation**: Jetpack Navigation Compose
- **Networking**: Retrofit + OkHttp (for non-core networking if needed)
- **Video Player**: ExoPlayer
- **Image Loading**: Coil
- **Local Storage**: DataStore (for settings), Room (if needed)

#### Gradle Dependencies
```kotlin
dependencies {
    // Kotlin
    implementation("org.jetbrains.kotlin:kotlin-stdlib:1.9.22")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.3")
    
    // Android Core
    implementation("androidx.core:core-ktx:1.12.0")
    implementation("androidx.lifecycle:lifecycle-runtime-ktx:2.7.0")
    
    // Compose
    implementation(platform("androidx.compose:compose-bom:2024.01.00"))
    implementation("androidx.compose.ui:ui")
    implementation("androidx.compose.material3:material3")
    implementation("androidx.compose.ui:ui-tooling-preview")
    implementation("androidx.activity:activity-compose:1.8.2")
    implementation("androidx.navigation:navigation-compose:2.7.6")
    
    // ViewModel
    implementation("androidx.lifecycle:lifecycle-viewmodel-compose:2.7.0")
    
    // Hilt
    implementation("com.google.dagger:hilt-android:2.50")
    kapt("com.google.dagger:hilt-compiler:2.50")
    implementation("androidx.hilt:hilt-navigation-compose:1.1.0")
    
    // ExoPlayer
    implementation("androidx.media3:media3-exoplayer:1.2.1")
    implementation("androidx.media3:media3-ui:1.2.1")
    implementation("androidx.media3:media3-common:1.2.1")
    
    // Image Loading
    implementation("io.coil-kt:coil-compose:2.5.0")
    
    // DataStore
    implementation("androidx.datastore:datastore-preferences:1.0.0")
    
    // Rust FFI (generated)
    implementation(files("libs/stremio-android-bridge.jar"))
}
```

### Build System

#### Android Build Configuration

**build.gradle.kts (app module)**:
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
        versionCode = 1
        versionName = "1.0.0"
        
        ndk {
            abiFilters.addAll(listOf("armeabi-v7a", "arm64-v8a", "x86", "x86_64"))
        }
    }
    
    buildFeatures {
        compose = true
    }
    
    composeOptions {
        kotlinCompilerExtensionVersion = "1.5.8"
    }
}

// Task to build Rust library
tasks.register<Exec>("buildRustLib") {
    workingDir = file("../stremio-android-bridge")
    commandLine = listOf("cargo", "ndk", "build", "--release")
}

tasks.whenTaskAdded {
    if (name == "preBuild") {
        dependsOn("buildRustLib")
    }
}
```

---

## Implementation Roadmap

### Detailed Steps for Getting Started

#### Step 1: Repository Setup

1. **Fork/Clone stremio-core**:
   ```bash
   git clone https://github.com/Stremio/stremio-core.git
   cd stremio-core
   ```

2. **Create Android bridge workspace member**:
   ```toml
   # Add to Cargo.toml
   [workspace]
   members = [
       "stremio-core-web", 
       "stremio-derive", 
       "stremio-watched-bitfield",
       "stremio-android-bridge"  # New member
   ]
   ```

3. **Create Android bridge directory**:
   ```bash
   cargo new --lib stremio-android-bridge
   cd stremio-android-bridge
   ```

#### Step 2: Install Required Tools

1. **Rust toolchain**:
   ```bash
   # Install Rust (if not already installed)
   curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
   
   # Add Android targets
   rustup target add \
       aarch64-linux-android \
       armv7-linux-androideabi \
       i686-linux-android \
       x86_64-linux-android
   ```

2. **cargo-ndk**:
   ```bash
   cargo install cargo-ndk
   ```

3. **uniffi-bindgen**:
   ```bash
   cargo install uniffi-bindgen
   ```

4. **Android NDK**:
   - Install via Android Studio SDK Manager
   - Or download from: https://developer.android.com/ndk/downloads
   - Set `ANDROID_NDK_HOME` environment variable

#### Step 3: Create Basic uniffi Bridge

**stremio-android-bridge/src/lib.rs**:
```rust
use stremio_core::types::profile::Profile;

// Simple example function to test FFI
#[uniffi::export]
pub fn hello_stremio() -> String {
    "Hello from Stremio Core!".to_string()
}

// More complex example
#[uniffi::export]
pub fn get_app_version() -> String {
    env!("CARGO_PKG_VERSION").to_string()
}

uniffi::include_scaffolding!("stremio_android");
```

**stremio-android-bridge/src/stremio_android.udl**:
```udl
namespace stremio_android {
    string hello_stremio();
    string get_app_version();
};
```

#### Step 4: Build Configuration

**stremio-android-bridge/Cargo.toml**:
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
uniffi = { version = "0.25" }

[build-dependencies]
uniffi = { version = "0.25", features = ["build"] }
```

**stremio-android-bridge/build.rs**:
```rust
fn main() {
    uniffi::generate_scaffolding("src/stremio_android.udl").unwrap();
}
```

#### Step 5: Build for Android

```bash
# Build for all Android architectures
cd stremio-android-bridge
cargo ndk -t armeabi-v7a -t arm64-v8a -t x86 -t x86_64 \
    -o ../android/app/src/main/jniLibs \
    build --release

# Generate Kotlin bindings
uniffi-bindgen generate src/stremio_android.udl \
    --language kotlin \
    --out-dir ../android/app/src/main/java
```

#### Step 6: Android Project Setup

1. **Create Android project** in Android Studio:
   - Language: Kotlin
   - Minimum SDK: API 24 (Android 7.0)
   - Use Jetpack Compose

2. **Configure JNI libraries**:
   - Place `.so` files in `app/src/main/jniLibs/`
   - Structure:
     ```
     jniLibs/
     ├── arm64-v8a/
     │   └── libstremio_android_bridge.so
     ├── armeabi-v7a/
     │   └── libstremio_android_bridge.so
     ├── x86/
     │   └── libstremio_android_bridge.so
     └── x86_64/
         └── libstremio_android_bridge.so
     ```

3. **Copy generated Kotlin bindings** to your project:
   ```kotlin
   // Generated by uniffi
   // Use it in your Kotlin code:
   import stremio_android.*
   
   val message = helloStremio()
   println(message) // "Hello from Stremio Core!"
   ```

---

## Key Components

### 1. Android Environment Implementation

The environment implementation is crucial as it provides platform-specific functionality to stremio-core.

**Key Responsibilities**:
- HTTP networking
- Local storage
- Addon transport
- Analytics (optional)

**Reference**: Study `stremio-core-web/src/env.rs` for patterns

**Implementation Sketch**:
```rust
pub struct AndroidEnv {
    http_client: Arc<AndroidHttpClient>,
    storage: Arc<AndroidStorage>,
}

impl Env for AndroidEnv {
    fn fetch<IN, OUT>(&self, request: Request<IN>) -> EnvFuture<OUT>
    where
        IN: Serialize,
        OUT: for<'de> Deserialize<'de> + 'static,
    {
        // Implement using Android's HTTP stack
        // Could use reqwest or call back to Kotlin
    }
    
    fn storage_get(&self, key: &str) -> EnvFuture<Option<String>> {
        // Use SharedPreferences or DataStore
    }
    
    fn storage_set(&self, key: &str, value: String) -> EnvFuture<()> {
        // Save to SharedPreferences or DataStore
    }
}
```

### 2. Runtime Management

The Runtime handles all state management and effects.

**Initialization**:
```rust
#[uniffi::export]
pub fn initialize_runtime(
    storage_data: Option<String>,
) -> Result<RuntimeHandle, Error> {
    let env = AndroidEnv::new();
    let runtime = Runtime::new(env, storage_data);
    Ok(RuntimeHandle::new(runtime))
}
```

**State Observation**:
```rust
#[uniffi::export]
pub fn observe_state(
    runtime: &RuntimeHandle,
    callback: Box<dyn StateCallback>,
) {
    // Set up observer that calls back to Kotlin
}
```

### 3. Model Bridge

Bridge individual models to Kotlin:

```rust
#[uniffi::export]
pub struct CtxBridge {
    runtime: Arc<RwLock<Runtime<AndroidEnv>>>,
}

#[uniffi::export]
impl CtxBridge {
    pub fn get_profile(&self) -> Option<ProfileData> {
        // Serialize profile for Kotlin
    }
    
    pub fn login(&self, email: String, password: String) {
        // Dispatch login action
    }
    
    pub fn install_addon(&self, transport_url: String) {
        // Dispatch install addon action
    }
}
```

### 4. Data Serialization

Convert Rust types to Kotlin-friendly types:

```rust
#[derive(uniffi::Record)]
pub struct ProfileData {
    pub id: String,
    pub email: Option<String>,
    pub avatar: Option<String>,
    pub settings: SettingsData,
}

// Implement From<Profile> for ProfileData
```

### 5. Action Dispatching

```rust
#[uniffi::export]
pub enum CoreAction {
    Login { email: String, password: String },
    Logout,
    InstallAddon { transport_url: String },
    OpenCatalog { catalog_id: String },
    // ... more actions
}

#[uniffi::export]
pub fn dispatch_action(
    runtime: &RuntimeHandle,
    action: CoreAction,
) -> Result<(), Error> {
    // Convert to stremio_core::Action and dispatch
}
```

---

## Integration Patterns

### Pattern 1: Unidirectional Data Flow

```
User Action (Kotlin)
  ↓
Dispatch Action to Runtime (Rust)
  ↓
Runtime processes Effect
  ↓
State Update
  ↓
Notify Observer (Rust → Kotlin)
  ↓
UI Update (Compose)
```

**Implementation**:

```kotlin
// ViewModel
class LibraryViewModel @Inject constructor(
    private val coreBridge: CoreBridge
) : ViewModel() {
    
    private val _libraryState = MutableStateFlow<LibraryState>(LibraryState.Loading)
    val libraryState: StateFlow<LibraryState> = _libraryState.asStateFlow()
    
    init {
        // Observe state from Rust core
        coreBridge.observeLibrary { rustState ->
            _libraryState.value = rustState.toKotlinState()
        }
    }
    
    fun addToLibrary(metaId: String) {
        coreBridge.dispatchAction(
            CoreAction.AddToLibrary(metaId)
        )
    }
}

// Composable
@Composable
fun LibraryScreen(viewModel: LibraryViewModel = hiltViewModel()) {
    val state by viewModel.libraryState.collectAsState()
    
    when (state) {
        is LibraryState.Loading -> LoadingView()
        is LibraryState.Loaded -> LibraryGrid(state.items)
        is LibraryState.Error -> ErrorView(state.message)
    }
}
```

### Pattern 2: Repository Pattern

Wrap the Rust bridge in a repository for cleaner architecture:

```kotlin
interface StremioRepository {
    fun observeLibrary(): Flow<Result<Library>>
    suspend fun addToLibrary(metaId: String): Result<Unit>
    suspend fun removeFromLibrary(metaId: String): Result<Unit>
}

class StremioRepositoryImpl @Inject constructor(
    private val coreBridge: CoreBridge
) : StremioRepository {
    
    override fun observeLibrary(): Flow<Result<Library>> = callbackFlow {
        val listener = coreBridge.observeLibrary { state ->
            trySend(Result.success(state.toLibrary()))
        }
        
        awaitClose { listener.cancel() }
    }
    
    override suspend fun addToLibrary(metaId: String): Result<Unit> {
        return withContext(Dispatchers.IO) {
            try {
                coreBridge.dispatchAction(CoreAction.AddToLibrary(metaId))
                Result.success(Unit)
            } catch (e: Exception) {
                Result.failure(e)
            }
        }
    }
}
```

### Pattern 3: Error Handling

```rust
#[derive(uniffi::Error, thiserror::Error, Debug)]
pub enum CoreError {
    #[error("Network error: {message}")]
    Network { message: String },
    
    #[error("Authentication error: {message}")]
    Auth { message: String },
    
    #[error("Not found: {message}")]
    NotFound { message: String },
    
    #[error("Internal error: {message}")]
    Internal { message: String },
}

// Convert from EnvError to CoreError
impl From<EnvError> for CoreError {
    fn from(error: EnvError) -> Self {
        match error {
            EnvError::Fetch(msg) => CoreError::Network { message: msg },
            EnvError::StorageReadError(msg) => CoreError::Internal { message: msg },
            // ... more conversions
            _ => CoreError::Internal { message: error.message() },
        }
    }
}
```

```kotlin
// Kotlin side handles errors gracefully
sealed class UiState<out T> {
    object Loading : UiState<Nothing>()
    data class Success<T>(val data: T) : UiState<T>()
    data class Error(val error: CoreError) : UiState<Nothing>()
}
```

---

## Testing Strategy

### Unit Tests

#### Rust Side
```rust
#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn test_android_env_storage() {
        let env = AndroidEnv::new_test();
        // Test storage operations
    }
    
    #[test]
    fn test_profile_serialization() {
        let profile = create_test_profile();
        let profile_data = ProfileData::from(profile);
        assert_eq!(profile_data.email, Some("test@example.com".to_string()));
    }
}
```

#### Kotlin Side
```kotlin
class CoreBridgeTest {
    @Test
    fun testHelloStremio() {
        val message = helloStremio()
        assertEquals("Hello from Stremio Core!", message)
    }
    
    @Test
    fun testActionDispatch() = runTest {
        val bridge = CoreBridge.initialize(null)
        bridge.dispatchAction(CoreAction.Login(
            email = "test@example.com",
            password = "password"
        ))
        // Verify state change
    }
}
```

### Integration Tests

Test the FFI boundary:
```kotlin
@RunWith(AndroidJUnit4::class)
class CoreIntegrationTest {
    
    @Test
    fun testFullAuthFlow() = runTest {
        val bridge = CoreBridge.initialize(null)
        
        // Test login
        bridge.login("user@example.com", "password")
        delay(1000)
        
        val profile = bridge.getProfile()
        assertNotNull(profile)
        assertEquals("user@example.com", profile.email)
    }
}
```

### UI Tests

Use Compose testing:
```kotlin
@Test
fun testLibraryScreen() {
    composeTestRule.setContent {
        LibraryScreen()
    }
    
    composeTestRule.onNodeWithText("My Library").assertExists()
    composeTestRule.onNodeWithTag("library-grid").assertExists()
}
```

### Performance Tests

Monitor FFI call overhead:
```kotlin
@Test
fun benchmarkFFICalls() {
    val iterations = 10000
    val startTime = System.nanoTime()
    
    repeat(iterations) {
        helloStremio()
    }
    
    val avgTime = (System.nanoTime() - startTime) / iterations
    assertTrue(avgTime < 10_000) // Less than 10 microseconds
}
```

---

## Deployment & Distribution

### Build Variants

```kotlin
android {
    buildTypes {
        debug {
            applicationIdSuffix = ".debug"
            isDebuggable = true
        }
        
        release {
            isMinifyEnabled = true
            isShrinkResources = true
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
            signingConfig = signingConfigs.getByName("release")
        }
    }
    
    flavorDimensions += "version"
    productFlavors {
        create("free") {
            dimension = "version"
            applicationIdSuffix = ".free"
        }
        create("pro") {
            dimension = "version"
        }
    }
}
```

### ProGuard Rules

**proguard-rules.pro**:
```proguard
# Keep uniffi-generated classes
-keep class uniffi.** { *; }
-keep class stremio_android.** { *; }

# Keep native methods
-keepclasseswithmembernames class * {
    native <methods>;
}

# Keep Rust FFI classes
-keep class com.stremio.android.bridge.** { *; }
```

### Continuous Integration

**GitHub Actions Example**:
```yaml
name: Android CI

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Rust
      uses: actions-rs/toolchain@v1
      with:
        toolchain: stable
        profile: minimal
        
    - name: Add Android targets
      run: |
        rustup target add aarch64-linux-android
        rustup target add armv7-linux-androideabi
        rustup target add i686-linux-android
        rustup target add x86_64-linux-android
        
    - name: Install cargo-ndk
      run: cargo install cargo-ndk
      
    - name: Build Rust library
      run: |
        cd stremio-android-bridge
        cargo ndk build --release
        
    - name: Set up JDK
      uses: actions/setup-java@v3
      with:
        java-version: '17'
        distribution: 'temurin'
        
    - name: Build Android app
      run: ./gradlew assembleDebug
      
    - name: Run tests
      run: ./gradlew test
      
    - name: Upload APK
      uses: actions/upload-artifact@v3
      with:
        name: app-debug
        path: app/build/outputs/apk/debug/app-debug.apk
```

### Play Store Release Checklist

- [ ] App signed with release key
- [ ] ProGuard enabled and tested
- [ ] All permissions justified in privacy policy
- [ ] Screenshots for all device types
- [ ] Feature graphic and app icon
- [ ] Short and full description
- [ ] Privacy policy URL
- [ ] Content rating completed
- [ ] Target API level meets requirements
- [ ] App bundle (.aab) generated
- [ ] Closed testing completed
- [ ] Open testing completed (optional)

---

## Additional Resources

### Documentation
- [Stremio Core Repository](https://github.com/Stremio/stremio-core)
- [uniffi-rs Guide](https://mozilla.github.io/uniffi-rs/)
- [Android NDK Guide](https://developer.android.com/ndk/guides)
- [Jetpack Compose Documentation](https://developer.android.com/jetpack/compose)
- [Rust Android Guide](https://mozilla.github.io/firefox-browser-architecture/experiments/2017-09-21-rust-on-android.html)

### Similar Projects
- Mozilla Firefox for Android (uses Rust)
- Signal Android (uses Rust for crypto)
- 1Password Android (uses Rust core)

### Community
- Stremio Discord/Forum
- Rust Android Working Group
- uniffi-rs Discussion Forum

---

## Conclusion

Building a Stremio Android app using the core Rust library is a substantial but achievable project. The key advantages are:

1. **Code Reuse**: Leverage battle-tested business logic
2. **Performance**: Rust provides excellent performance
3. **Maintainability**: Single source of truth for core logic
4. **Cross-platform**: Easier to maintain consistency across platforms

The estimated timeline is **6-7 months** for a full-featured app with a team of 2-3 developers (1-2 Android developers, 1 Rust developer). For a solo developer, expect 9-12 months.

### Success Factors

1. **Strong Rust knowledge**: Essential for bridge development
2. **Android expertise**: Modern Kotlin and Compose experience
3. **Understanding of FFI**: Critical for proper integration
4. **Reference implementation**: Study stremio-core-web extensively
5. **Incremental development**: Build and test features incrementally
6. **Community support**: Engage with Stremio and Rust communities

### Next Steps

1. Set up development environment (see Phase 1)
2. Create proof-of-concept with basic FFI call
3. Implement AndroidEnv (see Phase 2)
4. Build minimal UI with one feature
5. Iterate and expand features
6. Polish and release

Good luck with your Stremio Android app development!
