# Android Integration Documentation

This directory contains comprehensive documentation for building an Android application using the stremio-core Rust library.

## Documents Overview

### 📋 [ANDROID_APP_PLAN.md](./ANDROID_APP_PLAN.md)
**The Complete Guide** - A thorough, step-by-step plan for building a Stremio Android app.

**Contents:**
- Executive Summary
- Repository Overview
- Architecture Strategy (uniffi-rs, JNI, etc.)
- 8 Development Phases with detailed tasks
- Technical Stack (Rust + Kotlin)
- Implementation Roadmap
- Key Components
- Integration Patterns
- Testing Strategy
- Deployment & Distribution

**Who should read this:** Project managers, technical leads, and developers wanting a comprehensive overview of the entire project.

**Estimated Timeline:** 6-7 months with 2-3 developers

---

### 🚀 [QUICK_START.md](./QUICK_START.md)
**Get Started in Minutes** - A practical guide to set up your first Android-Rust integration.

**Contents:**
- Prerequisites and tool installation
- Step-by-step setup (13 steps)
- Creating the Android bridge
- Building for Android targets
- Generating Kotlin bindings
- Testing the integration
- Troubleshooting common issues

**Who should read this:** Developers ready to start coding and want to see "Hello World" from Rust on Android.

**Time to Complete:** 2-4 hours

---

### ⚙️ [ENVIRONMENT_IMPLEMENTATION.md](./ENVIRONMENT_IMPLEMENTATION.md)
**Deep Dive into Platform Integration** - Detailed guide for implementing the `Env` trait for Android.

**Contents:**
- Understanding the Env trait
- Architecture options (Pure Rust vs Callbacks)
- Complete AndroidEnv implementation
- HTTP client setup (reqwest)
- Storage implementation (sled database)
- Testing strategies
- Advanced topics (permissions, custom storage)
- Performance optimization

**Who should read this:** Developers working on the core integration between Rust and Android, especially those implementing platform-specific functionality.

**Difficulty:** Intermediate to Advanced

---

## Quick Navigation

### Getting Started?
1. Read the **Executive Summary** in [ANDROID_APP_PLAN.md](./ANDROID_APP_PLAN.md)
2. Follow [QUICK_START.md](./QUICK_START.md) to get a working integration
3. Return to [ANDROID_APP_PLAN.md](./ANDROID_APP_PLAN.md) for the complete roadmap

### Already Have Basic Integration?
1. Study [ENVIRONMENT_IMPLEMENTATION.md](./ENVIRONMENT_IMPLEMENTATION.md)
2. Implement the environment for Android
3. Proceed to Phase 3 in [ANDROID_APP_PLAN.md](./ANDROID_APP_PLAN.md)

### Planning the Project?
1. Read the full [ANDROID_APP_PLAN.md](./ANDROID_APP_PLAN.md)
2. Review the timeline and phases
3. Assess your team's skills and resources

## Architecture Overview

```
┌─────────────────────────────────────────────┐
│          Android App (Kotlin)                │
│  ┌────────────────────────────────────────┐ │
│  │   UI Layer (Jetpack Compose)           │ │
│  └────────────────┬───────────────────────┘ │
│  ┌────────────────▼───────────────────────┐ │
│  │   ViewModel Layer                       │ │
│  └────────────────┬───────────────────────┘ │
│  ┌────────────────▼───────────────────────┐ │
│  │   Kotlin Bindings (Generated)          │ │
│  └────────────────┬───────────────────────┘ │
└───────────────────┼──────────────────────────┘
                    │ JNI/FFI (uniffi)
┌───────────────────▼──────────────────────────┐
│   stremio-android-bridge (Rust)              │
│   - uniffi definitions                       │
│   - Android environment implementation       │
└───────────────────┬──────────────────────────┘
                    │
┌───────────────────▼──────────────────────────┐
│   stremio-core (Rust)                        │
│   - Core business logic                      │
│   - Models (Ctx, Library, Player, etc.)      │
└──────────────────────────────────────────────┘
```

## Key Technologies

### Rust Side
- **stremio-core**: Core business logic
- **uniffi-rs**: Automatic binding generation
- **tokio**: Async runtime
- **reqwest**: HTTP client
- **sled**: Embedded database

### Android Side
- **Kotlin**: Application language
- **Jetpack Compose**: UI framework
- **MVVM**: Architecture pattern
- **Hilt**: Dependency injection
- **ExoPlayer**: Video playback
- **Coroutines**: Async operations

## Development Phases Summary

| Phase | Duration | Focus |
|-------|----------|-------|
| 1. Foundation | 3 weeks | Setup environment, basic FFI |
| 2. Environment | 3 weeks | Implement AndroidEnv trait |
| 3. Core Integration | 3 weeks | Runtime and models |
| 4. Feature Implementation | 5 weeks | Core models (Ctx, Library, etc.) |
| 5. UI Development | 6 weeks | Build Android UI |
| 6. Platform Features | 3 weeks | Android-specific features |
| 7. Polish & Optimization | 3 weeks | Performance and testing |
| 8. Release Preparation | 2 weeks | App Store and deployment |
| **Total** | **28 weeks** | **~6-7 months** |

## Prerequisites

### Required Knowledge
- ✅ Rust programming (intermediate level)
- ✅ Kotlin/Android development
- ✅ Understanding of FFI/JNI concepts
- ✅ Async programming (Rust tokio, Kotlin coroutines)
- ✅ REST APIs and HTTP

### Tools Required
- Rust toolchain (1.77+)
- Android Studio
- Android NDK (25+)
- cargo-ndk
- uniffi-bindgen

See [QUICK_START.md](./QUICK_START.md) for installation instructions.

## Reference Implementation

The **stremio-core-web** implementation (located in `/stremio-core-web`) serves as an excellent reference for:
- How to implement the `Env` trait
- Model serialization patterns
- Runtime management
- Event handling

While it's WASM-specific, the patterns are directly applicable to Android.

## Common Patterns

### 1. State Observation
```kotlin
// Kotlin ViewModel
coreBridge.observeLibrary { state ->
    _libraryState.value = state.toKotlinState()
}
```

### 2. Action Dispatching
```kotlin
fun addToLibrary(metaId: String) {
    coreBridge.dispatchAction(
        CoreAction.AddToLibrary(metaId)
    )
}
```

### 3. Error Handling
```rust
// Rust
#[derive(uniffi::Error)]
pub enum CoreError {
    Network { message: String },
    Storage { message: String },
    // ...
}
```

```kotlin
// Kotlin
try {
    coreBridge.someOperation()
} catch (e: CoreError.Network) {
    // Handle network error
}
```

## Testing Strategy

### Unit Tests
- Rust: Test environment and bridge logic
- Kotlin: Test ViewModels and repositories

### Integration Tests
- Test FFI boundary
- Test full flow (Rust → Kotlin → UI)

### UI Tests
- Compose UI tests
- End-to-end user flows

See [ANDROID_APP_PLAN.md](./ANDROID_APP_PLAN.md#testing-strategy) for details.

## Performance Considerations

- **FFI Overhead**: Minimize boundary crossings
- **Serialization**: Use efficient formats
- **Threading**: Handle async properly on both sides
- **Memory**: Manage lifetimes carefully
- **Network**: Implement caching and retries

## Troubleshooting

Common issues are documented in:
- [QUICK_START.md - Troubleshooting](./QUICK_START.md#troubleshooting)
- [ENVIRONMENT_IMPLEMENTATION.md - Troubleshooting](./ENVIRONMENT_IMPLEMENTATION.md#troubleshooting)

## Getting Help

### Resources
- [Stremio Core Repository](https://github.com/Stremio/stremio-core)
- [uniffi-rs Documentation](https://mozilla.github.io/uniffi-rs/)
- [Android Developers](https://developer.android.com/)
- [Rust Book](https://doc.rust-lang.org/book/)

### Community
- Stremio Community Forums
- Rust Android Working Group
- uniffi-rs Discussions

## Contributing

If you find issues or have improvements:
1. Open an issue in the repository
2. Submit a pull request
3. Share your experience with the community

## License

This documentation is part of the stremio-core repository and follows the same license terms.

## Acknowledgments

- Stremio team for creating stremio-core
- Mozilla for uniffi-rs
- Rust and Android communities

---

**Ready to start?** Head to [QUICK_START.md](./QUICK_START.md) and build your first integration!
