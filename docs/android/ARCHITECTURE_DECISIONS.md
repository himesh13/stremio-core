# Architecture Decision Record: Android App Implementation

## Status
**PROPOSED** - This is a planning document for a new Android implementation

## Context

We want to create an Android application that mimics the Stremio Android app functionality. The stremio-core repository contains all the core business logic written in Rust, designed to be reusable across platforms. We need to decide on the best approach for integrating this Rust code with an Android application.

## Decision Drivers

1. **Code Reuse**: Maximize reuse of existing stremio-core logic
2. **Performance**: Ensure the app performs well on Android devices
3. **Maintainability**: Keep the codebase maintainable and easy to update
4. **Development Speed**: Enable rapid development and iteration
5. **Type Safety**: Maintain type safety across the FFI boundary
6. **Native Experience**: Provide a native Android user experience
7. **Team Skills**: Consider the team's expertise in Rust and Android

## Options Considered

### Option 1: Pure Rust with Native UI (REJECTED)

**Description**: Write the entire app in Rust, including UI, using frameworks like Druid or egui.

**Pros**:
- Maximum code reuse
- Single language
- Potential for code sharing with other platforms

**Cons**:
- ❌ UI frameworks are immature
- ❌ No native Android look and feel
- ❌ Poor integration with Android platform features
- ❌ Limited tooling and IDE support
- ❌ Difficult to hire Android developers who know Rust

**Decision**: REJECTED - Native Android UI is essential for a good user experience.

---

### Option 2: React Native + WASM (REJECTED)

**Description**: Use React Native for UI and compile stremio-core to WebAssembly.

**Pros**:
- Cross-platform UI (iOS + Android)
- JavaScript/TypeScript is widely known
- Existing stremio-core-web can be reused

**Cons**:
- ❌ WASM performance overhead
- ❌ Bridge overhead (JS ↔ WASM)
- ❌ Not truly native
- ❌ React Native has its own complexity
- ❌ Larger app size

**Decision**: REJECTED - Performance concerns and not native enough.

---

### Option 3: Flutter + flutter_rust_bridge (CONSIDERED)

**Description**: Use Flutter for UI and flutter_rust_bridge for Rust integration.

**Pros**:
- ✅ Modern UI framework
- ✅ Good Rust integration
- ✅ Cross-platform (iOS + Android)
- ✅ flutter_rust_bridge is well-maintained

**Cons**:
- ⚠️ Not native Android widgets
- ⚠️ Requires learning Flutter/Dart
- ⚠️ Flutter apps are larger
- ⚠️ Specific to Flutter ecosystem

**Decision**: CONSIDERED but not chosen - Good option if you already use Flutter.

---

### Option 4: Native Android (Kotlin) + Manual JNI (REJECTED)

**Description**: Build UI in Kotlin with Jetpack Compose, use manual JNI for Rust integration.

**Pros**:
- ✅ Full control over FFI
- ✅ Native Android experience
- ✅ Modern Kotlin + Compose

**Cons**:
- ❌ Manual JNI is tedious and error-prone
- ❌ High maintenance burden
- ❌ Type safety issues
- ❌ Lots of boilerplate code

**Decision**: REJECTED - Too much manual work, high risk of bugs.

---

### Option 5: Native Android (Kotlin) + uniffi-rs (SELECTED ✅)

**Description**: Build UI in Kotlin with Jetpack Compose, use Mozilla's uniffi-rs for automatic binding generation.

**Pros**:
- ✅ Native Android experience
- ✅ Automatic binding generation (less error-prone)
- ✅ Type-safe FFI
- ✅ Well-maintained by Mozilla
- ✅ Used in production (Firefox, etc.)
- ✅ Clean, idiomatic Kotlin API
- ✅ Good documentation
- ✅ Supports callbacks and async

**Cons**:
- ⚠️ Some learning curve for uniffi
- ⚠️ Limited to uniffi-supported types
- ⚠️ Android-specific implementation needed

**Decision**: **SELECTED** - Best balance of all factors.

---

## Detailed Architecture: Selected Option

### High-Level Architecture

```
┌─────────────────────────────────────┐
│   UI Layer (Jetpack Compose)        │
│   - Screens & Navigation            │
│   - Composables                     │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│   ViewModel Layer                   │
│   - State management                │
│   - Business logic coordination     │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│   Repository Layer                  │
│   - Data source abstraction         │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│   Kotlin Bindings (Generated)       │
│   - uniffi generated code           │
│   - Type-safe FFI                   │
└──────────────┬──────────────────────┘
               │ JNI
┌──────────────▼──────────────────────┐
│   stremio-android-bridge (Rust)     │
│   - AndroidEnv implementation       │
│   - uniffi exports                  │
│   - Platform adapters               │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│   stremio-core (Rust)               │
│   - Core business logic             │
│   - Platform-agnostic code          │
└─────────────────────────────────────┘
```

### Technology Stack

**Android/Kotlin Side:**
- Language: Kotlin
- UI: Jetpack Compose
- Architecture: MVVM
- DI: Hilt
- Async: Coroutines + Flow
- Video: ExoPlayer
- Images: Coil

**Rust Side:**
- Core: stremio-core
- FFI: uniffi-rs
- Async: tokio
- HTTP: reqwest
- Storage: sled or rocksdb
- Logging: tracing

### Key Design Decisions

#### 1. Environment Implementation: Pure Rust

**Decision**: Implement the `Env` trait in pure Rust using Android-compatible crates.

**Rationale**:
- Better performance (fewer FFI crossings)
- Type safety
- Easier testing
- Portable implementation

**Trade-off**: Less direct access to Android APIs, but reqwest and sled work well.

#### 2. State Management: Unidirectional Data Flow

**Decision**: Use unidirectional data flow with Kotlin StateFlow.

**Pattern**:
```
User Action → Dispatch to Rust → Process → Update State → Notify Kotlin → Update UI
```

**Rationale**:
- Predictable state updates
- Easy to debug
- Works well with Compose
- Matches stremio-core's architecture

#### 3. Serialization: JSON via serde

**Decision**: Use JSON for data passing across FFI boundary.

**Rationale**:
- Human-readable for debugging
- Well-supported in both Rust and Kotlin
- stremio-core already uses serde
- Performance is acceptable for this use case

**Alternative Considered**: Protocol Buffers (more efficient but adds complexity)

#### 4. Error Handling: Typed Errors

**Decision**: Use uniffi error types for structured error handling.

**Example**:
```rust
#[derive(uniffi::Error)]
pub enum CoreError {
    Network { message: String },
    Storage { message: String },
    Auth { message: String },
}
```

**Rationale**:
- Type-safe error handling in Kotlin
- Clear error categories
- Easy to handle specific errors in UI

#### 5. Threading Model: Tokio + Coroutines

**Decision**: Use tokio runtime in Rust, Kotlin coroutines in Android.

**Pattern**:
- Rust: async/await with tokio
- FFI boundary: Synchronous calls (uniffi limitation)
- Kotlin: Wrap in coroutines with Dispatchers.IO

**Rationale**:
- Natural async handling on both sides
- uniffi doesn't support async yet (planned)
- Coroutines can wrap synchronous FFI calls

#### 6. Storage: Embedded Database (sled)

**Decision**: Use sled embedded database for Rust-side storage.

**Rationale**:
- Pure Rust implementation
- No external dependencies
- Good performance
- ACID transactions
- Works on Android

**Alternative Considered**: Room database via callbacks (more complex)

## Consequences

### Positive

- ✅ Clean separation of concerns
- ✅ Maximum code reuse from stremio-core
- ✅ Native Android look and feel
- ✅ Type-safe FFI with minimal boilerplate
- ✅ Good performance characteristics
- ✅ Maintainable architecture
- ✅ Clear upgrade path for stremio-core updates

### Negative

- ⚠️ Learning curve for uniffi-rs
- ⚠️ Two languages to maintain (Rust + Kotlin)
- ⚠️ FFI boundary requires careful memory management
- ⚠️ Limited async support in uniffi (workaround needed)
- ⚠️ Build process is more complex

### Risks and Mitigations

| Risk | Mitigation |
|------|------------|
| uniffi breaking changes | Pin uniffi version, test upgrades carefully |
| FFI performance issues | Profile early, minimize boundary crossings |
| Memory leaks | Use Arc/Rc properly, test with leak detection |
| Difficult debugging | Extensive logging on both sides |
| Complex build process | Document thoroughly, use CI/CD |

## Implementation Plan

See [ANDROID_APP_PLAN.md](./ANDROID_APP_PLAN.md) for the complete 8-phase implementation plan.

**Quick Summary**:
1. Foundation (3 weeks) - Setup, basic FFI
2. Environment (3 weeks) - AndroidEnv implementation
3. Core Integration (3 weeks) - Runtime and models
4. Features (5 weeks) - Core models implementation
5. UI (6 weeks) - Compose UI
6. Platform Features (3 weeks) - Android-specific features
7. Polish (3 weeks) - Testing and optimization
8. Release (2 weeks) - Deployment

**Total**: ~6-7 months with 2-3 developers

## Validation

This decision should be validated by:

1. **Proof of Concept**: Build a minimal app with one feature (Week 1-2)
2. **Performance Testing**: Measure FFI overhead (Week 3)
3. **Developer Feedback**: Team's comfort with the stack (Week 4)

## References

- [uniffi-rs Documentation](https://mozilla.github.io/uniffi-rs/)
- [stremio-core Repository](https://github.com/Stremio/stremio-core)
- [stremio-core-web Implementation](https://github.com/Stremio/stremio-core/tree/master/stremio-core-web)
- [Android NDK Guide](https://developer.android.com/ndk/guides)
- [Jetpack Compose](https://developer.android.com/jetpack/compose)

## Decision

We will proceed with **Option 5: Native Android (Kotlin) + uniffi-rs**.

**Approved by**: [To be filled]  
**Date**: [To be filled]  
**Review Date**: After Phase 1 completion (Week 3)

---

## Appendix: Comparison Matrix

| Criteria | Option 1<br>(Pure Rust) | Option 2<br>(RN + WASM) | Option 3<br>(Flutter) | Option 4<br>(Manual JNI) | Option 5<br>(uniffi) |
|----------|----------|----------|----------|----------|----------|
| Native Feel | ❌ Low | ⚠️ Medium | ⚠️ Medium | ✅ High | ✅ High |
| Performance | ✅ High | ❌ Low | ✅ High | ✅ High | ✅ High |
| Dev Speed | ❌ Slow | ✅ Fast | ✅ Fast | ❌ Slow | ✅ Fast |
| Type Safety | ✅ High | ❌ Low | ✅ High | ⚠️ Medium | ✅ High |
| Maintenance | ⚠️ Medium | ⚠️ Medium | ✅ Good | ❌ Poor | ✅ Good |
| Team Skills | ⚠️ Rare | ✅ Common | ⚠️ Medium | ✅ Common | ✅ Common |
| Tooling | ❌ Poor | ✅ Good | ✅ Good | ✅ Excellent | ✅ Good |

Legend: ✅ Excellent | ⚠️ Good | ❌ Poor

---

*This document follows the [Architecture Decision Records](https://adr.github.io/) format.*
