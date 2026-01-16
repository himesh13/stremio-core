# Android Environment Implementation Guide

This guide provides detailed instructions for implementing the `Env` trait for Android, which is essential for integrating stremio-core with an Android application.

## Overview

The `Env` trait is the bridge between stremio-core's platform-agnostic code and platform-specific functionality. It provides:

- **HTTP networking** for API calls and addon communication
- **Local storage** for persisting user data
- **Addon transport** for communicating with Stremio addons
- **Analytics** (optional) for usage tracking

## Understanding the Env Trait

Located in `src/runtime/env.rs`, the `Env` trait defines the interface:

```rust
pub trait Env: Clone + 'static {
    fn fetch<IN, OUT>(&self, request: Request<IN>) -> EnvFuture<OUT>
    where
        IN: Serialize,
        OUT: for<'de> Deserialize<'de> + 'static;

    fn fetch_serde<OUT>(&self, request: Request<()>) -> EnvFuture<OUT>
    where
        OUT: for<'de> Deserialize<'de> + 'static;

    fn addon_transport(&self, transport_url: &Url) -> Box<dyn AddonTransport>;

    fn analytics_context(&self) -> &AnalyticsContext;

    fn storage_get(&self, key: &str) -> EnvFuture<Option<String>>;
    fn storage_set(&self, key: &str, value: String) -> EnvFuture<()>;
}
```

## Architecture Options

### Option 1: Pure Rust Implementation (Recommended)

Implement everything in Rust using Android-compatible crates.

**Advantages**:
- Type safety
- Better performance
- Fewer FFI crossings

**Disadvantages**:
- Limited to Rust HTTP/storage libraries
- May need to handle Android-specific permissions from Rust

### Option 2: Callback to Kotlin

Use uniffi callbacks to delegate operations to Kotlin.

**Advantages**:
- Use native Android APIs
- Easier permission handling
- Access to Android-specific features

**Disadvantages**:
- More FFI overhead
- Complex callback management
- Potential thread safety issues

We recommend **Option 1** for most cases, with callbacks only for permission-sensitive operations.

## Implementation: Pure Rust Approach

### Step 1: Set Up Dependencies

Add to `stremio-android-bridge/Cargo.toml`:

```toml
[dependencies]
# Core dependencies
stremio-core = { path = "../", features = ["derive", "env-future-send"] }
uniffi = "0.25"

# Async runtime
tokio = { version = "1", features = ["rt", "rt-multi-thread", "sync", "time"] }

# HTTP client
reqwest = { version = "0.11", features = ["json", "rustls-tls"], default-features = false }

# Storage
sled = "0.34"  # Embedded database
# OR
rocksdb = "0.21"  # Alternative

# Serialization
serde = { version = "1", features = ["derive"] }
serde_json = "1.0"

# Error handling
anyhow = "1.0"
thiserror = "1.0"

# Logging
tracing = "0.1"
tracing-android = "0.1"
tracing-subscriber = "0.3"

# URL handling
url = "2.4"

# Other
futures = "0.3"
once_cell = "1.0"
parking_lot = "0.12"
```

### Step 2: Create AndroidEnv Structure

Create `stremio-android-bridge/src/env.rs`:

```rust
use std::sync::Arc;
use anyhow::Result;
use parking_lot::RwLock;
use reqwest::Client;
use sled::Db;
use stremio_core::{
    addon_transport::{AddonHTTPTransport, AddonTransport},
    runtime::{Env, EnvError, EnvFuture},
};
use tracing::{debug, error, info, warn};
use url::Url;

#[derive(Clone)]
pub struct AndroidEnv {
    http_client: Arc<Client>,
    storage: Arc<AndroidStorage>,
    storage_path: Arc<str>,
}

impl AndroidEnv {
    /// Create a new AndroidEnv with the given storage path
    pub fn new(storage_path: impl Into<String>) -> Result<Self> {
        let storage_path: Arc<str> = storage_path.into().into();
        
        // Create HTTP client with sensible defaults
        let http_client = Arc::new(
            Client::builder()
                .timeout(std::time::Duration::from_secs(30))
                .user_agent(concat!(
                    "StremioAndroid/",
                    env!("CARGO_PKG_VERSION")
                ))
                .build()
                .map_err(|e| anyhow::anyhow!("Failed to create HTTP client: {}", e))?
        );

        // Create storage
        let storage = Arc::new(AndroidStorage::new(&storage_path)?);

        info!("AndroidEnv initialized with storage at: {}", storage_path);

        Ok(Self {
            http_client,
            storage,
            storage_path,
        })
    }
}

/// Storage implementation using sled embedded database
pub struct AndroidStorage {
    db: Db,
}

impl AndroidStorage {
    pub fn new(path: impl AsRef<std::path::Path>) -> Result<Self> {
        let db = sled::open(path)
            .map_err(|e| anyhow::anyhow!("Failed to open database: {}", e))?;
        
        Ok(Self { db })
    }

    pub fn get(&self, key: &str) -> Result<Option<String>> {
        let value = self.db
            .get(key.as_bytes())
            .map_err(|e| anyhow::anyhow!("Failed to read from storage: {}", e))?;
        
        match value {
            Some(bytes) => {
                let string = String::from_utf8(bytes.to_vec())
                    .map_err(|e| anyhow::anyhow!("Invalid UTF-8 in storage: {}", e))?;
                Ok(Some(string))
            }
            None => Ok(None),
        }
    }

    pub fn set(&self, key: &str, value: &str) -> Result<()> {
        self.db
            .insert(key.as_bytes(), value.as_bytes())
            .map_err(|e| anyhow::anyhow!("Failed to write to storage: {}", e))?;
        
        self.db
            .flush()
            .map_err(|e| anyhow::anyhow!("Failed to flush storage: {}", e))?;
        
        Ok(())
    }

    pub fn remove(&self, key: &str) -> Result<()> {
        self.db
            .remove(key.as_bytes())
            .map_err(|e| anyhow::anyhow!("Failed to remove from storage: {}", e))?;
        
        self.db
            .flush()
            .map_err(|e| anyhow::anyhow!("Failed to flush storage: {}", e))?;
        
        Ok(())
    }
}
```

### Step 3: Implement the Env Trait

Continue in `env.rs`:

```rust
impl Env for AndroidEnv {
    fn fetch<IN, OUT>(&self, request: http::Request<IN>) -> EnvFuture<OUT>
    where
        IN: serde::Serialize,
        OUT: for<'de> serde::Deserialize<'de> + 'static,
    {
        let http_client = self.http_client.clone();
        
        Box::pin(async move {
            // Convert http::Request to reqwest::Request
            let (parts, body) = request.into_parts();
            
            let url = parts.uri.to_string();
            let method = parts.method;
            
            debug!("Fetching: {} {}", method, url);
            
            // Build reqwest request
            let mut req_builder = http_client
                .request(method.clone(), &url);
            
            // Add headers
            for (name, value) in parts.headers.iter() {
                if let Ok(value_str) = value.to_str() {
                    req_builder = req_builder.header(name.as_str(), value_str);
                }
            }
            
            // Add body if present
            let body_json = serde_json::to_string(&body)
                .map_err(|e| EnvError::Serde(e.to_string()))?;
            
            if !body_json.is_empty() && body_json != "null" {
                req_builder = req_builder
                    .header("Content-Type", "application/json")
                    .body(body_json);
            }
            
            // Execute request
            let response = req_builder
                .send()
                .await
                .map_err(|e| EnvError::Fetch(format!("Request failed: {}", e)))?;
            
            // Check status
            let status = response.status();
            if !status.is_success() {
                return Err(EnvError::Fetch(format!(
                    "HTTP error: {} - {}",
                    status,
                    response.text().await.unwrap_or_default()
                )));
            }
            
            // Parse response
            let text = response
                .text()
                .await
                .map_err(|e| EnvError::Fetch(format!("Failed to read response: {}", e)))?;
            
            debug!("Response received: {} bytes", text.len());
            
            serde_json::from_str(&text)
                .map_err(|e| EnvError::Serde(format!("Failed to parse response: {}", e)))
        })
    }

    fn fetch_serde<OUT>(&self, request: http::Request<()>) -> EnvFuture<OUT>
    where
        OUT: for<'de> serde::Deserialize<'de> + 'static,
    {
        // Delegate to fetch with empty body
        self.fetch::<(), OUT>(request)
    }

    fn addon_transport(&self, transport_url: &Url) -> Box<dyn AddonTransport> {
        // Use HTTP transport for addons
        Box::new(AddonHTTPTransport::new(
            transport_url.clone(),
            self.http_client.clone(),
        ))
    }

    fn analytics_context(&self) -> &stremio_core::analytics::AnalyticsContext {
        // Return empty analytics context (or implement your own)
        static ANALYTICS: once_cell::sync::Lazy<stremio_core::analytics::AnalyticsContext> =
            once_cell::sync::Lazy::new(|| stremio_core::analytics::AnalyticsContext {
                // Configure analytics if needed
                ..Default::default()
            });
        
        &ANALYTICS
    }

    fn storage_get(&self, key: &str) -> EnvFuture<Option<String>> {
        let storage = self.storage.clone();
        let key = key.to_string();
        
        Box::pin(async move {
            debug!("Storage get: {}", key);
            
            storage
                .get(&key)
                .map_err(|e| EnvError::StorageReadError(e.to_string()))
        })
    }

    fn storage_set(&self, key: &str, value: String) -> EnvFuture<()> {
        let storage = self.storage.clone();
        let key = key.to_string();
        
        Box::pin(async move {
            debug!("Storage set: {} ({} bytes)", key, value.len());
            
            storage
                .set(&key, &value)
                .map_err(|e| EnvError::StorageWriteError(e.to_string()))
        })
    }
}
```

### Step 4: Export via uniffi

Update `src/stremio_android.udl`:

```udl
namespace stremio_android {
    [Throws=CoreError]
    AndroidEnv create_env(string storage_path);
};

[Error]
enum CoreError {
    "EnvCreation",
    "Storage",
    "Network",
    "Serialization",
    "Other",
};
```

Update `src/lib.rs`:

```rust
mod env;

use env::AndroidEnv;

#[uniffi::export]
pub fn create_env(storage_path: String) -> Result<AndroidEnv, CoreError> {
    AndroidEnv::new(storage_path)
        .map_err(|e| CoreError::EnvCreation { 
            message: e.to_string() 
        })
}

#[derive(uniffi::Error, thiserror::Error, Debug)]
pub enum CoreError {
    #[error("Failed to create environment: {message}")]
    EnvCreation { message: String },
    
    #[error("Storage error: {message}")]
    Storage { message: String },
    
    #[error("Network error: {message}")]
    Network { message: String },
    
    #[error("Serialization error: {message}")]
    Serialization { message: String },
    
    #[error("Other error: {message}")]
    Other { message: String },
}

uniffi::include_scaffolding!("stremio_android");
```

## Testing the Environment

### Rust Tests

Add to `src/env.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn test_storage() {
        let env = AndroidEnv::new("/tmp/test_storage").unwrap();
        
        // Test set
        env.storage_set("test_key", "test_value".to_string())
            .await
            .unwrap();
        
        // Test get
        let value = env.storage_get("test_key").await.unwrap();
        assert_eq!(value, Some("test_value".to_string()));
    }

    #[tokio::test]
    async fn test_fetch() {
        let env = AndroidEnv::new("/tmp/test_storage").unwrap();
        
        // Test HTTP GET
        let request = http::Request::builder()
            .method("GET")
            .uri("https://httpbin.org/get")
            .body(())
            .unwrap();
        
        let response: serde_json::Value = env.fetch_serde(request).await.unwrap();
        assert!(response.is_object());
    }
}
```

Run tests:
```bash
cargo test --package stremio-android-bridge
```

### Kotlin Integration Test

```kotlin
@Test
fun testEnvironmentCreation() = runTest {
    val storagePath = context.filesDir.absolutePath + "/stremio"
    val env = createEnv(storagePath)
    assertNotNull(env)
}
```

## Advanced Topics

### Handling Android Permissions

For network and storage access, ensure proper permissions in `AndroidManifest.xml`:

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" 
    android:maxSdkVersion="28" />
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"
    android:maxSdkVersion="32" />
```

### Using Android's HTTP Client (Callback Approach)

If you prefer using Android's native HTTP client:

```rust
// Define callback trait in UDL
pub trait HttpClient: Send + Sync {
    fn fetch(&self, url: String, method: String) -> Result<String, String>;
}

// Use in AndroidEnv
pub struct AndroidEnv {
    http_client: Arc<dyn HttpClient>,
    // ... other fields
}
```

Then implement in Kotlin:
```kotlin
class AndroidHttpClient : HttpClient {
    override fun fetch(url: String, method: String): String {
        // Use OkHttp or HttpURLConnection
        val client = OkHttpClient()
        val request = Request.Builder()
            .url(url)
            .method(method, null)
            .build()
        
        val response = client.newCall(request).execute()
        return response.body?.string() ?: ""
    }
}
```

### Custom Storage Solutions

#### Using DataStore (Kotlin)

```kotlin
class DataStoreStorage(private val dataStore: DataStore<Preferences>) {
    suspend fun get(key: String): String? {
        return dataStore.data.first()[stringPreferencesKey(key)]
    }
    
    suspend fun set(key: String, value: String) {
        dataStore.edit { preferences ->
            preferences[stringPreferencesKey(key)] = value
        }
    }
}
```

#### Using Room Database

For complex queries, use Room and expose via callbacks.

### Performance Optimization

1. **Connection Pooling**: reqwest handles this automatically
2. **Caching**: Implement HTTP cache
3. **Batch Operations**: Use batch storage operations when possible
4. **Compression**: Enable gzip in HTTP client

```rust
let http_client = Client::builder()
    .pool_max_idle_per_host(10)
    .gzip(true)
    .timeout(Duration::from_secs(30))
    .build()?;
```

## Next Steps

With the environment implemented, you can now:

1. Initialize the Runtime with AndroidEnv
2. Start implementing models (Ctx, Library, etc.)
3. Handle events and state updates
4. Build the UI layer

See the main [ANDROID_APP_PLAN.md](./ANDROID_APP_PLAN.md) for the complete roadmap.

## Troubleshooting

### Issue: Async runtime errors

**Solution**: Ensure tokio runtime is properly initialized:
```rust
#[uniffi::export]
pub fn initialize_tokio() {
    tokio::runtime::Runtime::new().unwrap();
}
```

### Issue: Storage permission denied

**Solution**: Request storage permissions at runtime (Android 6.0+):
```kotlin
if (ContextCompat.checkSelfPermission(this, Manifest.permission.WRITE_EXTERNAL_STORAGE)
    != PackageManager.PERMISSION_GRANTED) {
    ActivityCompat.requestPermissions(this,
        arrayOf(Manifest.permission.WRITE_EXTERNAL_STORAGE),
        REQUEST_CODE)
}
```

### Issue: Network requests fail

**Solution**: Check:
- INTERNET permission in manifest
- Network security config for cleartext traffic
- TLS certificates are valid

## Reference Implementation

Study `stremio-core-web/src/env.rs` for a complete example of environment implementation, though it's WASM-specific, the patterns are similar.
