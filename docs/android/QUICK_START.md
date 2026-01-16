# Quick Start Guide: Android App Development with Stremio Core

This guide will help you get started quickly with building an Android app using the stremio-core Rust library.

## Prerequisites

### Required Tools

1. **Rust** (stable, version 1.77+)
   ```bash
   curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
   source $HOME/.cargo/env
   ```

2. **Android Studio** (latest stable version)
   - Download from: https://developer.android.com/studio

3. **Android NDK** (version 25 or later)
   - Install via Android Studio: Tools → SDK Manager → SDK Tools → NDK

4. **cargo-ndk**
   ```bash
   cargo install cargo-ndk
   ```

5. **uniffi-bindgen**
   ```bash
   cargo install uniffi-bindgen
   ```

### Environment Setup

Set environment variables:

```bash
# Add to ~/.bashrc or ~/.zshrc
export ANDROID_HOME=$HOME/Android/Sdk
export ANDROID_NDK_HOME=$ANDROID_HOME/ndk/25.2.9519653  # Adjust version
export PATH=$PATH:$ANDROID_HOME/platform-tools:$ANDROID_HOME/tools
```

Add Android targets to Rust:

```bash
rustup target add aarch64-linux-android armv7-linux-androideabi x86_64-linux-android i686-linux-android
```

## Step-by-Step Setup

### Step 1: Clone the Repository

```bash
git clone https://github.com/Stremio/stremio-core.git
cd stremio-core
```

### Step 2: Create Android Bridge Crate

Create a new workspace member for the Android bridge:

```bash
cargo new --lib stremio-android-bridge
cd stremio-android-bridge
```

### Step 3: Configure Cargo.toml

Edit `stremio-android-bridge/Cargo.toml`:

```toml
[package]
name = "stremio-android-bridge"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib"]
name = "stremio_android_bridge"

[dependencies]
stremio-core = { path = "../", features = ["derive"] }
uniffi = "0.25"
serde = { version = "1", features = ["derive"] }
serde_json = "1.0"
tokio = { version = "1", features = ["rt", "sync"] }
tracing = "0.1"
tracing-android = "0.1"
anyhow = "1.0"

[build-dependencies]
uniffi = { version = "0.25", features = ["build"] }
```

Update root `Cargo.toml` to include the new member:

```toml
[workspace]
members = [
    "stremio-core-web", 
    "stremio-derive", 
    "stremio-watched-bitfield",
    "stremio-android-bridge"
]
```

### Step 4: Create uniffi Interface Definition

Create `stremio-android-bridge/src/stremio_android.udl`:

```udl
namespace stremio_android {
    string hello_stremio();
    string get_version();
};
```

### Step 5: Implement Basic Bridge

Edit `stremio-android-bridge/src/lib.rs`:

```rust
// Simple test function
#[uniffi::export]
pub fn hello_stremio() -> String {
    "Hello from Stremio Core Rust!".to_string()
}

#[uniffi::export]
pub fn get_version() -> String {
    env!("CARGO_PKG_VERSION").to_string()
}

// Include the generated scaffolding
uniffi::include_scaffolding!("stremio_android");
```

### Step 6: Create Build Script

Create `stremio-android-bridge/build.rs`:

```rust
fn main() {
    uniffi::generate_scaffolding("src/stremio_android.udl")
        .expect("Failed to generate uniffi scaffolding");
}
```

### Step 7: Build for Android

```bash
cd stremio-android-bridge

# Build for all Android architectures
cargo ndk \
    -t armeabi-v7a \
    -t arm64-v8a \
    -t x86 \
    -t x86_64 \
    -o ./jniLibs \
    build --release
```

This will create shared libraries in `./jniLibs/`:
- `arm64-v8a/libstremio_android_bridge.so`
- `armeabi-v7a/libstremio_android_bridge.so`
- `x86/libstremio_android_bridge.so`
- `x86_64/libstremio_android_bridge.so`

### Step 8: Generate Kotlin Bindings

```bash
uniffi-bindgen generate \
    src/stremio_android.udl \
    --language kotlin \
    --out-dir ./kotlin
```

This generates Kotlin bindings in `./kotlin/`.

### Step 9: Create Android Project

Open Android Studio and create a new project:
- **Template**: Empty Activity (Compose)
- **Name**: StremioAndroid
- **Package name**: com.stremio.android
- **Language**: Kotlin
- **Minimum SDK**: API 24 (Android 7.0)

### Step 10: Copy Native Libraries

Copy the generated `.so` files:

```bash
# From stremio-core directory
cp -r stremio-android-bridge/jniLibs/* \
    android/StremioAndroid/app/src/main/jniLibs/
```

Directory structure should be:
```
app/src/main/jniLibs/
├── arm64-v8a/
│   └── libstremio_android_bridge.so
├── armeabi-v7a/
│   └── libstremio_android_bridge.so
├── x86/
│   └── libstremio_android_bridge.so
└── x86_64/
    └── libstremio_android_bridge.so
```

### Step 11: Copy Kotlin Bindings

Copy generated Kotlin files:

```bash
cp -r stremio-android-bridge/kotlin/uniffi \
    android/StremioAndroid/app/src/main/java/
```

### Step 12: Configure Android App

Edit `app/build.gradle.kts`:

```kotlin
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
}

android {
    namespace = "com.stremio.android"
    compileSdk = 34

    defaultConfig {
        applicationId = "com.stremio.android"
        minSdk = 24
        targetSdk = 34
        versionCode = 1
        versionName = "1.0"

        ndk {
            // Specify which ABIs to include
            abiFilters.addAll(listOf("arm64-v8a", "armeabi-v7a"))
        }
    }

    buildTypes {
        release {
            isMinifyEnabled = false
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }

    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }

    kotlinOptions {
        jvmTarget = "17"
    }

    buildFeatures {
        compose = true
    }

    composeOptions {
        kotlinCompilerExtensionVersion = "1.5.8"
    }

    packaging {
        resources {
            excludes += "/META-INF/{AL2.0,LGPL2.1}"
        }
        
        // Important: Don't compress .so files
        jniLibs {
            useLegacyPackaging = false
        }
    }
}

dependencies {
    implementation("androidx.core:core-ktx:1.12.0")
    implementation("androidx.lifecycle:lifecycle-runtime-ktx:2.7.0")
    implementation("androidx.activity:activity-compose:1.8.2")
    implementation(platform("androidx.compose:compose-bom:2024.01.00"))
    implementation("androidx.compose.ui:ui")
    implementation("androidx.compose.ui:ui-graphics")
    implementation("androidx.compose.ui:ui-tooling-preview")
    implementation("androidx.compose.material3:material3")
    
    // JNA for uniffi (add to dependencies if needed)
    implementation("net.java.dev.jna:jna:5.13.0@aar")
}
```

### Step 13: Test the Integration

Create a test activity:

```kotlin
package com.stremio.android

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import uniffi.stremio_android.*

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            MaterialTheme {
                Surface(
                    modifier = Modifier.fillMaxSize(),
                    color = MaterialTheme.colorScheme.background
                ) {
                    TestScreen()
                }
            }
        }
    }
}

@Composable
fun TestScreen() {
    var message by remember { mutableStateOf("") }
    var version by remember { mutableStateOf("") }

    LaunchedEffect(Unit) {
        try {
            // Call Rust functions
            message = helloStremio()
            version = getVersion()
        } catch (e: Exception) {
            message = "Error: ${e.message}"
        }
    }

    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Text(
            text = message,
            style = MaterialTheme.typography.headlineMedium
        )
        Spacer(modifier = Modifier.height(16.dp))
        Text(
            text = "Version: $version",
            style = MaterialTheme.typography.bodyLarge
        )
    }
}
```

### Step 14: Build and Run

1. Connect an Android device or start an emulator
2. Click "Run" in Android Studio
3. You should see "Hello from Stremio Core Rust!" on the screen

## Troubleshooting

### Common Issues

#### Issue: "Library not found" error

**Solution**: Make sure `.so` files are in the correct directory and NDK ABIs match.

```kotlin
// In build.gradle.kts, check:
ndk {
    abiFilters.addAll(listOf("arm64-v8a", "armeabi-v7a"))
}
```

#### Issue: uniffi-bindgen not found

**Solution**: Install it globally:
```bash
cargo install uniffi-bindgen
```

#### Issue: Build fails with "NDK not found"

**Solution**: Set `ANDROID_NDK_HOME`:
```bash
export ANDROID_NDK_HOME=$ANDROID_HOME/ndk/25.2.9519653
```

Or configure in `local.properties`:
```properties
ndk.dir=/Users/username/Library/Android/sdk/ndk/25.2.9519653
```

#### Issue: Kotlin can't find uniffi classes

**Solution**: Ensure JNA dependency is added:
```kotlin
implementation("net.java.dev.jna:jna:5.13.0@aar")
```

#### Issue: App crashes on start

**Solution**: Check logcat for native library loading errors. Common causes:
- Missing ABI in `abiFilters`
- Wrong library name
- Missing JNA dependency

View logs:
```bash
adb logcat | grep -i "stremio\|uniffi\|native"
```

### Debugging Tips

1. **Enable verbose logging**:
   ```kotlin
   // Add to MainActivity
   System.setProperty("jna.debug_load", "true")
   ```

2. **Check library loading**:
   ```kotlin
   try {
       System.loadLibrary("stremio_android_bridge")
       Log.d("Native", "Library loaded successfully")
   } catch (e: UnsatisfiedLinkError) {
       Log.e("Native", "Failed to load library", e)
   }
   ```

3. **Verify APK contents**:
   ```bash
   # Extract APK
   unzip -l app/build/outputs/apk/debug/app-debug.apk | grep ".so"
   ```

## Next Steps

Now that you have a working integration:

1. **Implement AndroidEnv**: See `docs/android/ENVIRONMENT_IMPLEMENTATION.md`
2. **Add Runtime**: Integrate stremio-core Runtime
3. **Build Features**: Start implementing models (Ctx, Library, etc.)
4. **Create UI**: Build Compose screens

Refer to the main [ANDROID_APP_PLAN.md](./ANDROID_APP_PLAN.md) for detailed implementation phases.

## Helpful Resources

- [uniffi-rs Guide](https://mozilla.github.io/uniffi-rs/)
- [Android NDK Documentation](https://developer.android.com/ndk/guides)
- [Jetpack Compose Tutorial](https://developer.android.com/jetpack/compose/tutorial)
- [Stremio Core API Docs](https://stremio.github.io/stremio-core)

## Getting Help

If you run into issues:
1. Check the troubleshooting section above
2. Review the example code in this repository
3. Consult the stremio-core-web implementation as reference
4. Ask in Stremio community forums/Discord
