# Frequently Asked Questions (FAQ)

## General Questions

### Q: Why use Rust for an Android app?

**A**: The core business logic of Stremio is written in Rust in the `stremio-core` library. By using this library:
- ✅ You get battle-tested, platform-agnostic business logic
- ✅ Share code with other Stremio platforms (Web, iOS, Desktop)
- ✅ Benefit from Rust's performance and safety guarantees
- ✅ Reduce maintenance burden (single source of truth)

The Android app only needs to implement UI and platform integration—the hard work is already done!

### Q: How mature is this approach?

**A**: Very mature. Production apps using Rust+Android:
- **Firefox for Android** - Uses Rust extensively via uniffi
- **Signal** - Uses Rust for cryptography
- **1Password** - Uses Rust core across platforms
- **Dropbox** - Uses Rust for sync engine

The tooling (uniffi-rs, cargo-ndk) is well-maintained and used in production.

### Q: What's the learning curve?

**A**: 
- **If you know Rust and Android**: 1-2 weeks to be productive
- **If you know only Android**: 2-4 weeks (need to learn Rust basics and FFI concepts)
- **If you know only Rust**: 2-4 weeks (need to learn Android/Kotlin)
- **If you're new to both**: 2-3 months (learn both stacks first)

The good news: You don't need to be a Rust expert. Most of the core logic is already implemented.

### Q: What's the estimated development time?

**A**: For a full-featured app similar to official Stremio Android:
- **Team of 2-3 developers**: 6-7 months
- **Solo developer**: 9-12 months
- **Proof of concept**: 2-3 weeks

See [ANDROID_APP_PLAN.md](./ANDROID_APP_PLAN.md) for detailed timeline.

## Technical Questions

### Q: Why uniffi-rs instead of manual JNI?

**A**: 
- ✅ **Type safety**: uniffi generates type-safe bindings automatically
- ✅ **Less code**: No manual JNI boilerplate
- ✅ **Fewer bugs**: Automatic generation reduces human error
- ✅ **Better DX**: Idiomatic Kotlin code, not JNI mess
- ✅ **Maintained**: Mozilla actively maintains it

Manual JNI is tedious, error-prone, and hard to maintain.

### Q: Can I use Flutter instead of native Android?

**A**: Yes! You can use Flutter with `flutter_rust_bridge` instead of uniffi. The architecture would be similar:

```
Flutter UI → flutter_rust_bridge → stremio-android-bridge → stremio-core
```

Pros:
- ✅ Cross-platform UI (iOS + Android)
- ✅ Fast development

Cons:
- ⚠️ Not native widgets
- ⚠️ Larger app size

See [ARCHITECTURE_DECISIONS.md](./ARCHITECTURE_DECISIONS.md) for comparison.

### Q: What about iOS?

**A**: The same approach works for iOS! Use uniffi-rs to generate Swift bindings. The architecture is identical:

```
SwiftUI → uniffi Swift bindings → stremio-ios-bridge → stremio-core
```

You'd implement an `IOSEnv` similar to `AndroidEnv`.

### Q: How do I handle async operations?

**A**: 
- **Rust side**: Use `async/await` with tokio
- **FFI boundary**: Currently synchronous (uniffi limitation)
- **Kotlin side**: Wrap FFI calls in coroutines with `Dispatchers.IO`

Example:
```kotlin
suspend fun loadLibrary(): Result<Library> = withContext(Dispatchers.IO) {
    try {
        val libraryData = coreBridge.getLibrary() // Blocking FFI call
        Result.success(libraryData.toKotlin())
    } catch (e: Exception) {
        Result.failure(e)
    }
}
```

uniffi is working on async support, which will simplify this.

### Q: How do I debug across the FFI boundary?

**A**:
1. **Rust side**: Use `tracing` crate with Android logger
2. **Kotlin side**: Use standard Android logging
3. **Both sides**: Add extensive logging at FFI boundary
4. **Tools**: 
   - Android Studio debugger for Kotlin
   - `lldb` for Rust (via ndk-gdb)

Example Rust logging setup:
```rust
use tracing_android::AndroidLayer;
use tracing_subscriber::layer::SubscriberExt;

fn init_logging() {
    let subscriber = tracing_subscriber::registry()
        .with(AndroidLayer::new("StremioCore"));
    tracing::subscriber::set_global_default(subscriber).unwrap();
}
```

### Q: What about performance?

**A**: FFI has minimal overhead:
- **FFI call overhead**: ~10-100 nanoseconds
- **Serialization**: Use efficient formats (bincode > JSON)
- **Strategy**: Minimize boundary crossings, batch operations

For a media app, FFI performance is not a bottleneck. Network I/O and UI rendering are the main bottlenecks.

### Q: How do I handle memory management?

**A**: uniffi handles most of it automatically:
- Rust objects are exposed via `Arc<T>` (reference counting)
- Kotlin holds a reference, Rust cleans up when no refs remain
- No manual memory management needed

Key principle: **Don't pass large data frequently**. Instead:
- Pass IDs and fetch details on-demand
- Use observers for state changes
- Cache on Kotlin side if needed

### Q: Can I use stremio-core-web bindings?

**A**: No, stremio-core-web uses WASM bindings which only work in browsers. You need to create separate Android bindings.

However, stremio-core-web is an excellent **reference implementation**. Study it to understand:
- How to implement the `Env` trait
- How to wrap models
- How to handle events
- How to structure the bridge

## Implementation Questions

### Q: Where do I start?

**A**: Follow this path:
1. Read [QUICK_START.md](./QUICK_START.md) (2-4 hours)
2. Get "Hello World" working
3. Implement `AndroidEnv` following [ENVIRONMENT_IMPLEMENTATION.md](./ENVIRONMENT_IMPLEMENTATION.md)
4. Integrate one model (e.g., Ctx)
5. Build basic UI for that model
6. Iterate and expand

### Q: Which model should I implement first?

**A**: Recommended order:
1. **Context (Ctx)** - Authentication and addons (required for everything)
2. **Library** - User's library (relatively simple)
3. **Catalog** - Browse content (more complex)
4. **MetaDetails** - View content details
5. **Player** - Playback state
6. **StreamingServer** - Optional, for local streaming

Start with Ctx because it's foundational.

### Q: How do I implement the Env trait?

**A**: Two approaches:

**Option 1: Pure Rust (Recommended)**
- Use `reqwest` for HTTP
- Use `sled` or `rocksdb` for storage
- Implement everything in Rust

**Option 2: Callback to Kotlin**
- Define callback traits in uniffi
- Implement in Kotlin using Android APIs
- Call back from Rust

See [ENVIRONMENT_IMPLEMENTATION.md](./ENVIRONMENT_IMPLEMENTATION.md) for detailed guide.

### Q: How do I handle Android permissions?

**A**: Declare in `AndroidManifest.xml`:
```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
```

For runtime permissions (Android 6.0+), request in Kotlin:
```kotlin
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
    requestPermissions(arrayOf(
        Manifest.permission.WRITE_EXTERNAL_STORAGE
    ), REQUEST_CODE)
}
```

### Q: How do I integrate ExoPlayer?

**A**: ExoPlayer is separate from stremio-core. The flow is:

```
1. User selects stream → Dispatch action to Rust
2. Rust returns stream URLs → Callback to Kotlin  
3. Kotlin passes URL to ExoPlayer → Start playback
4. Playback events → Update Rust state via actions
```

Example:
```kotlin
// Get stream URL from Rust
val streamUrl = coreBridge.getStreamUrl(metaId, videoId)

// Create ExoPlayer
val player = ExoPlayer.Builder(context).build()
val mediaItem = MediaItem.fromUri(streamUrl)
player.setMediaItem(mediaItem)
player.prepare()
player.play()

// Update progress
player.addListener(object : Player.Listener {
    override fun onPlaybackStateChanged(state: Int) {
        coreBridge.updatePlayerState(state)
    }
})
```

### Q: How do I handle addon installation?

**A**: Addons are managed by stremio-core's `Ctx` model:

```kotlin
// Kotlin side
suspend fun installAddon(transportUrl: String) {
    coreBridge.dispatchAction(
        CoreAction.InstallAddon(transportUrl)
    )
}

// Rust side implements the logic
```

stremio-core handles:
- Fetching addon manifest
- Validating addon
- Storing addon in profile
- Making requests to addon

You just need to:
- Provide UI to enter addon URL
- Call the action
- Display installed addons

## Build & Deployment Questions

### Q: How do I build for release?

**A**:
```bash
# 1. Build Rust in release mode
cd stremio-android-bridge
cargo ndk \
    -t armeabi-v7a \
    -t arm64-v8a \
    -o ../android/app/src/main/jniLibs \
    build --release

# 2. Build Android release
cd ../android
./gradlew assembleRelease

# Output: app/build/outputs/apk/release/app-release.apk
```

### Q: How do I optimize APK size?

**A**:
1. **Enable ProGuard/R8**:
   ```kotlin
   buildTypes {
       release {
           isMinifyEnabled = true
           isShrinkResources = true
       }
   }
   ```

2. **Use app bundles** (.aab instead of .apk):
   ```bash
   ./gradlew bundleRelease
   ```

3. **Limit ABIs**: Only include arm64-v8a and armeabi-v7a (drops x86/x86_64):
   ```kotlin
   ndk {
       abiFilters.addAll(listOf("arm64-v8a", "armeabi-v7a"))
   }
   ```

4. **Optimize Rust**: Already done with `opt-level = 's'` in Cargo.toml

5. **Use split APKs**: Let Play Store generate per-ABI APKs

Expected sizes:
- Debug APK: 50-80 MB
- Release APK: 20-40 MB
- Per-ABI release: 10-20 MB

### Q: How do I publish to Play Store?

**A**:
1. Create Play Store account ($25 one-time fee)
2. Generate signing key:
   ```bash
   keytool -genkey -v -keystore stremio-release.keystore \
       -alias stremio -keyalg RSA -keysize 2048 -validity 10000
   ```
3. Configure signing in `build.gradle.kts`
4. Build release bundle: `./gradlew bundleRelease`
5. Upload to Play Console
6. Fill in store listing (screenshots, description, etc.)
7. Submit for review

See [ANDROID_APP_PLAN.md - Phase 8](./ANDROID_APP_PLAN.md#phase-8-release-preparation-weeks-27-28) for checklist.

## Troubleshooting Questions

### Q: Build fails with "NDK not found"

**A**: Set `ANDROID_NDK_HOME`:
```bash
export ANDROID_NDK_HOME=$ANDROID_HOME/ndk/25.2.9519653
```

Or in `local.properties`:
```properties
ndk.dir=/path/to/android/sdk/ndk/25.2.9519653
```

### Q: App crashes with "UnsatisfiedLinkError"

**A**: Library loading failed. Check:
1. **.so files present** in `jniLibs/[abi]/`
2. **ABI matches device**: `adb shell getprop ro.product.cpu.abi`
3. **Library name correct**: Should be `libstremio_android_bridge.so`
4. **JNA dependency added**:
   ```kotlin
   implementation("net.java.dev.jna:jna:5.13.0@aar")
   ```

View logcat:
```bash
adb logcat | grep -i "unsatisfied\|stremio\|uniffi"
```

### Q: uniffi generation fails

**A**: Common issues:
1. **uniffi-bindgen not installed**: `cargo install uniffi-bindgen`
2. **Version mismatch**: uniffi crate and uniffi-bindgen versions must match
3. **UDL syntax error**: Check `.udl` file for typos
4. **Missing Cargo feature**: Ensure `uniffi` has `build` feature in build-dependencies

### Q: Kotlin can't find uniffi-generated classes

**A**:
1. Ensure bindings are generated: `uniffi-bindgen generate ...`
2. Check output directory: Should be in `app/src/main/java/uniffi/`
3. Sync Gradle: File → Sync Project with Gradle Files
4. Check imports: `import uniffi.stremio_android.*`

### Q: Async operations block the UI

**A**: Make sure you're using coroutines:
```kotlin
// ❌ Bad - Blocks UI thread
fun loadData() {
    val data = coreBridge.getData() // Blocking!
}

// ✅ Good - Non-blocking
suspend fun loadData() = withContext(Dispatchers.IO) {
    coreBridge.getData()
}

// ✅ Better - With proper error handling
suspend fun loadData(): Result<Data> = withContext(Dispatchers.IO) {
    try {
        Result.success(coreBridge.getData())
    } catch (e: Exception) {
        Result.failure(e)
    }
}
```

## Architecture Questions

### Q: Should I use MVVM, MVI, or another pattern?

**A**: **MVVM is recommended** because:
- ✅ Works well with Compose
- ✅ Clear separation of concerns
- ✅ ViewModel handles Rust bridge
- ✅ Widely understood pattern

Structure:
```
View (Composable) → ViewModel → Repository → CoreBridge → Rust
```

### Q: How do I handle configuration changes?

**A**: ViewModel survives configuration changes automatically. Store state there:

```kotlin
class HomeViewModel @Inject constructor(
    private val repository: StremioRepository
) : ViewModel() {
    
    // Survives rotation, etc.
    private val _state = MutableStateFlow<HomeState>(HomeState.Loading)
    val state: StateFlow<HomeState> = _state.asStateFlow()
    
    init {
        viewModelScope.launch {
            repository.observeHome().collect { result ->
                _state.value = result.toState()
            }
        }
    }
}
```

### Q: How do I handle deep links?

**A**: stremio-core has deep link generation. For handling:

1. **Declare in AndroidManifest.xml**:
```xml
<intent-filter android:autoVerify="true">
    <action android:name="android.intent.action.VIEW" />
    <category android:name="android.intent.category.DEFAULT" />
    <category android:name="android.intent.category.BROWSABLE" />
    <data android:scheme="stremio" />
</intent-filter>
```

2. **Handle in Activity**:
```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    
    intent?.data?.let { uri ->
        handleDeepLink(uri)
    }
}

fun handleDeepLink(uri: Uri) {
    // Parse URI and dispatch action to Rust
    coreBridge.openDeepLink(uri.toString())
}
```

## Community & Support Questions

### Q: Where can I get help?

**A**:
1. **This documentation**: Start here
2. **Stremio Community**: Discord, Forums
3. **uniffi-rs**: GitHub Discussions
4. **Rust Forums**: users.rust-lang.org
5. **Android Developers**: Stack Overflow

### Q: Can I contribute to stremio-core?

**A**: Yes! stremio-core is open source. Contributions welcome:
- Bug fixes
- New features
- Documentation
- Tests

See the main repository for contribution guidelines.

### Q: Is there example code?

**A**: 
- **stremio-core-web**: Reference implementation (WASM)
- **This documentation**: Code examples throughout
- **Official Stremio apps**: Closed source, but you can learn from behavior

### Q: What if I get stuck?

**A**:
1. Check [QUICK_START.md - Troubleshooting](./QUICK_START.md#troubleshooting)
2. Review [ENVIRONMENT_IMPLEMENTATION.md - Troubleshooting](./ENVIRONMENT_IMPLEMENTATION.md#troubleshooting)
3. Search existing issues in stremio-core repo
4. Ask in community forums
5. Open an issue with reproducible example

## Licensing Questions

### Q: Can I release my app commercially?

**A**: Check stremio-core's license (see LICENSE.md in the repository). Generally:
- ✅ You can use it in your app
- ✅ You can modify it
- ⚠️ Check if you need to attribute
- ⚠️ Check if you need to open-source your modifications

**Not legal advice** - consult a lawyer for commercial projects.

### Q: Can I publish to Play Store?

**A**: Yes, as long as you comply with:
1. stremio-core's license
2. Play Store policies
3. Content licensing laws in your jurisdiction

## Performance & Optimization Questions

### Q: How do I profile performance?

**A**:
- **Android**: Use Android Studio Profiler
- **Rust**: Use `cargo flamegraph`
- **FFI**: Add timing logs at boundary

```kotlin
// Profile FFI calls
val start = System.nanoTime()
val result = coreBridge.operation()
val duration = System.nanoTime() - start
Log.d("Performance", "Operation took ${duration / 1_000_000}ms")
```

### Q: How do I reduce memory usage?

**A**:
1. **Avoid large data copies** across FFI
2. **Stream data** when possible
3. **Use pagination** for lists
4. **Cache on Kotlin side** to reduce FFI calls
5. **Monitor with Android Profiler**

### Q: My app is slow. What do I check?

**A**: Common bottlenecks:
1. **Network requests**: Use caching, retry logic
2. **UI rendering**: Optimize Compose recompositions
3. **FFI calls**: Batch operations, reduce crossings
4. **Image loading**: Use Coil with proper sizing
5. **Database queries**: Use indexes, optimize queries

Profile to identify the actual bottleneck before optimizing!

---

## Still Have Questions?

- Read the complete [ANDROID_APP_PLAN.md](./ANDROID_APP_PLAN.md)
- Try the [QUICK_START.md](./QUICK_START.md) guide
- Check [ENVIRONMENT_IMPLEMENTATION.md](./ENVIRONMENT_IMPLEMENTATION.md)
- Review [ARCHITECTURE_DECISIONS.md](./ARCHITECTURE_DECISIONS.md)
- Ask in the community forums

**Good luck building your Stremio Android app! 🚀**
