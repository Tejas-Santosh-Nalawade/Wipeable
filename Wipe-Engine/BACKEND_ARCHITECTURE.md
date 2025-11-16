# Wipeable Backend Architecture & Implementation Details

## Backend System Architecture

The Wipeable backend is designed as a modular, cross-platform system with clear separation of concerns and robust error handling. Here's the detailed breakdown of the backend implementation:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                            BACKEND ARCHITECTURE                                 │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────────┐ │
│  │                          APPLICATION LAYER                                  │ │
│  │  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────────────┐ │ │
│  │  │   Entry Point   │───►│   Command       │───►│    Configuration        │ │ │
│  │  │   (main.rs)     │    │   Dispatcher    │    │    Management           │ │ │
│  │  └─────────────────┘    └─────────────────┘    └─────────────────────────┘ │ │
│  └─────────────────────────────────────────────────────────────────────────────┘ │
│                                      │                                           │
│                                      ▼                                           │
│  ┌─────────────────────────────────────────────────────────────────────────────┐ │
│  │                        SERVICE LAYER                                        │ │
│  │  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────────────┐ │ │
│  │  │  Wipe Service   │    │  Device Service │    │   Scheme Repository     │ │ │
│  │  │  (actions/)     │◄──►│  (storage/)     │◄──►│   (sanitization/)      │ │ │
│  │  └─────────────────┘    └─────────────────┘    └─────────────────────────┘ │ │
│  └─────────────────────────────────────────────────────────────────────────────┘ │
│                                      │                                           │
│                                      ▼                                           │
│  ┌─────────────────────────────────────────────────────────────────────────────┐ │
│  │                        BUSINESS LOGIC LAYER                                 │ │
│  │  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────────────┐ │ │
│  │  │ Task Execution  │    │ Pattern Engine  │    │   Error Handling        │ │ │
│  │  │ State Machine   │    │ & Verification  │    │   & Recovery            │ │ │
│  │  └─────────────────┘    └─────────────────┘    └─────────────────────────┘ │ │
│  └─────────────────────────────────────────────────────────────────────────────┘ │
│                                      │                                           │
│                                      ▼                                           │
│  ┌─────────────────────────────────────────────────────────────────────────────┐ │
│  │                         DATA ACCESS LAYER                                   │ │
│  │  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────────────┐ │ │
│  │  │  Platform       │    │   I/O Buffer    │    │   Bad Block             │ │ │
│  │  │  Abstraction    │    │   Management    │    │   Tracking              │ │ │
│  │  └─────────────────┘    └─────────────────┘    └─────────────────────────┘ │ │
│  └─────────────────────────────────────────────────────────────────────────────┘ │
│                                      │                                           │
│                                      ▼                                           │
│  ┌─────────────────────────────────────────────────────────────────────────────┐ │
│  │                        PLATFORM LAYER                                       │ │
│  │  ┌─────────────────┐                       ┌─────────────────────────────┐ │ │
│  │  │    Unix/Nix     │                       │        Windows              │ │ │
│  │  │  Implementation │                       │     Implementation          │ │ │
│  │  └─────────────────┘                       └─────────────────────────────┘ │ │
│  └─────────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────────┘
```

## Core Backend Components

### 1. Application Layer

#### Entry Point (`main.rs`)
```rust
// Application bootstrap and configuration
fn main() -> Result<()> {
    // Initialize logging and error handling
    // Parse command line arguments with clap
    // Setup storage enumeration
    // Route to appropriate command handler
    // Handle global error scenarios
}
```

**Key Responsibilities:**
- Command line parsing and validation
- Global error handling and reporting
- Application lifecycle management
- Resource cleanup and shutdown

#### Command Dispatcher
```rust
match app.get_matches_mut().subcommand() {
    Some(("list", _)) => handle_list_command(),
    Some(("wipe", cmd)) => handle_wipe_command(cmd),
    _ => show_help_and_exit(),
}
```

### 2. Service Layer

#### Wipe Service (`actions/wipe.rs`)

**Core Structures:**
```rust
pub struct WipeTask {
    pub scheme: Scheme,           // Sanitization scheme
    pub verify: Verify,           // Verification mode
    pub total_size: u64,          // Target size
    pub block_size: usize,        // I/O block size
    pub offset: u64,              // Starting offset
}

pub struct WipeState {
    pub stage: usize,             // Current stage
    pub at_verification: bool,    // Verification phase
    pub position: u64,            // Current position
    pub retries_left: u32,        // Retry attempts
    pub bad_blocks: Rc<RefCell<dyn BlockMarker>>,
}
```

**Key Operations:**
- Task creation and validation
- State management across stages  
- Progress tracking and reporting
- Error recovery and retry logic

#### Device Service (`storage/`)

**Platform Abstraction:**
```rust
pub trait StorageDevice {
    fn access(&self) -> Result<Box<dyn StorageAccess>>;
}

pub trait StorageAccess {
    fn position(&mut self) -> Result<u64>;
    fn seek(&mut self, position: u64) -> Result<u64>;
    fn read(&mut self, buffer: &mut [u8]) -> Result<usize>;
    fn write(&mut self, data: &[u8]) -> Result<()>;
    fn flush(&mut self) -> Result<()>;
}
```

**Platform Implementations:**
- **Unix/Linux** (`storage/nix/linux.rs`): Direct device access via file descriptors
- **macOS** (`storage/nix/macos.rs`): Core Foundation integration
- **Windows** (`storage/windows/`): WinAPI with proper access control

### 3. Business Logic Layer

#### Task Execution Engine

**State Machine Implementation:**
```rust
pub fn run(&self, access: &mut dyn StorageAccess, 
          state: &mut WipeState, 
          session: &mut dyn WipeEventReceiver) -> bool {
    
    // Stage iteration loop
    for stage_index in 0..self.scheme.stages.len() {
        // Execute sanitization stage
        if !self.execute_stage(stage, access, state, session) {
            return false;
        }
        
        // Optional verification stage
        if self.should_verify(stage_index) {
            if !self.verify_stage(stage, access, state, session) {
                return false;
            }
        }
    }
    true
}
```

#### Pattern Generation Engine (`sanitization/mem.rs`)

**Random Pattern Generator:**
```rust
pub struct RandomMemoryProvider {
    rng: ChaCha20Rng,
}

impl MemoryProvider for RandomMemoryProvider {
    fn fill(&mut self, buffer: &mut [u8]) {
        self.rng.fill_bytes(buffer);
    }
}
```

**Constant Pattern Generator:**
```rust
pub struct ConstantMemoryProvider {
    pattern: u8,
}

impl MemoryProvider for ConstantMemoryProvider {
    fn fill(&mut self, buffer: &mut [u8]) {
        buffer.fill(self.pattern);
    }
}
```

### 4. Data Access Layer

#### I/O Buffer Management

**Buffer Pool Implementation:**
```rust
pub struct BufferManager {
    buffer: Vec<u8>,
    size: usize,
}

impl BufferManager {
    pub fn new(block_size: usize) -> Self {
        BufferManager {
            buffer: vec![0u8; block_size],
            size: block_size,
        }
    }
    
    pub fn get_write_buffer(&mut self) -> &mut [u8] {
        &mut self.buffer[..self.size]
    }
    
    pub fn get_read_buffer(&mut self) -> &mut [u8] {
        &mut self.buffer[..self.size]
    }
}
```

#### Bad Block Tracking (`actions/marker.rs`)

**Roaring Bitmap Implementation:**
```rust
pub struct RoaringBlockMarker {
    bitmap: RoaringBitmap<u32>,
}

impl BlockMarker for RoaringBlockMarker {
    fn mark(&mut self, block: u64) {
        self.bitmap.insert(block as u32);
    }
    
    fn contains(&self, block: u64) -> bool {
        self.bitmap.contains(block as u32)
    }
}
```

## Platform-Specific Backend Implementation

### Unix/Linux Backend (`storage/nix/linux.rs`)

```rust
pub struct LinuxStorageDevice {
    path: PathBuf,
    details: StorageDetails,
}

impl StorageDevice for LinuxStorageDevice {
    fn access(&self) -> Result<Box<dyn StorageAccess>> {
        let file = OpenOptions::new()
            .read(true)
            .write(true)
            .open(&self.path)?;
            
        Ok(Box::new(LinuxStorageAccess { 
            file,
            position: 0,
        }))
    }
}
```

**Key Features:**
- Direct `/dev/` device access
- `sysfs` integration for device enumeration
- POSIX-compliant I/O operations
- Signal handling for cleanup

### Windows Backend (`storage/windows/`)

```rust
pub struct WindowsStorageDevice {
    path: String,
    details: StorageDetails,
}

impl StorageDevice for WindowsStorageDevice {
    fn access(&self) -> Result<Box<dyn StorageAccess>> {
        let handle = unsafe {
            CreateFileW(
                path_wide.as_ptr(),
                GENERIC_READ | GENERIC_WRITE,
                FILE_SHARE_READ | FILE_SHARE_WRITE,
                null_mut(),
                OPEN_EXISTING,
                FILE_FLAG_NO_BUFFERING | FILE_FLAG_WRITE_THROUGH,
                null_mut(),
            )
        };
        
        Ok(Box::new(WindowsStorageAccess { handle }))
    }
}
```

**Key Features:**
- `\\.\PhysicalDrive` access
- WMI integration for device properties
- Unbuffered I/O with `FILE_FLAG_NO_BUFFERING`
- Proper handle cleanup

### macOS Backend (`storage/nix/macos.rs`)

```rust
pub struct MacOSStorageDevice {
    path: PathBuf,
    details: StorageDetails,
}

// Uses Core Foundation for device enumeration
fn enumerate_devices() -> Result<Vec<StorageRef>> {
    // IOKit integration for device discovery
    // Disk Arbitration framework for properties
    // Security framework for permissions
}
```

**Key Features:**
- IOKit framework integration
- Disk Arbitration for device management
- Security framework for authorization
- BSD device node access

## Backend Configuration & Tuning

### Memory Management Strategy

```rust
pub struct MemoryConfiguration {
    pub block_size: usize,
    pub buffer_count: usize,
    pub use_mlock: bool,        // Lock pages in memory
    pub zero_on_free: bool,     // Security cleanup
}
```

### I/O Configuration

```rust
pub struct IOConfiguration {
    pub use_direct_io: bool,    // Bypass OS cache
    pub sync_after_write: bool, // Force disk sync
    pub verify_writes: bool,    // Read-back verification
    pub retry_count: u32,       // Error retry attempts
    pub retry_delay: Duration,  // Backoff delay
}
```

### Security Configuration

```rust
pub struct SecurityConfiguration {
    pub secure_memory: bool,    // Use secure allocators
    pub clear_buffers: bool,    // Zero buffers on free
    pub verify_patterns: bool,  // Validate generated patterns
    pub audit_logging: bool,    // Log security events
}
```

## Error Handling Architecture

### Error Type Hierarchy

```rust
#[derive(Error, Debug)]
pub enum LetheError {
    #[error("Storage operation failed: {0}")]
    Storage(#[from] StorageError),
    
    #[error("Sanitization failed: {0}")]
    Sanitization(#[from] SanitizationError),
    
    #[error("Configuration error: {0}")]
    Configuration(String),
    
    #[error("Permission denied: {0}")]
    Permission(String),
}
```

### Error Recovery Strategy

```rust
impl WipeTask {
    fn handle_error(&self, error: &StorageError, 
                   state: &mut WipeState) -> RecoveryAction {
        match error {
            StorageError::BadBlock => {
                state.bad_blocks.mark(current_block);
                RecoveryAction::Skip
            },
            StorageError::TemporaryFailure => {
                if state.retries_left > 0 {
                    state.retries_left -= 1;
                    RecoveryAction::Retry
                } else {
                    RecoveryAction::MarkBad
                }
            },
            StorageError::Fatal => RecoveryAction::Abort,
        }
    }
}
```

## Performance Optimization Backend

### Async I/O Considerations

```rust
// Current: Synchronous I/O for simplicity and security
// Future: Async I/O for performance improvements

pub struct AsyncWipeTask {
    // Tokio-based async implementation
    // Concurrent read/write operations
    // Background verification
}
```

### Memory Pool Management

```rust
pub struct MemoryPool {
    buffers: Vec<Vec<u8>>,
    available: VecDeque<usize>,
    block_size: usize,
}

impl MemoryPool {
    pub fn acquire_buffer(&mut self) -> Option<Vec<u8>> {
        self.available.pop_front()
            .map(|idx| std::mem::take(&mut self.buffers[idx]))
    }
    
    pub fn release_buffer(&mut self, mut buffer: Vec<u8>) {
        // Security: Zero buffer before reuse
        buffer.fill(0);
        let idx = self.buffers.len();
        self.buffers.push(buffer);
        self.available.push_back(idx);
    }
}
```

## Monitoring & Observability Backend

### Metrics Collection

```rust
pub struct WipeMetrics {
    pub bytes_processed: u64,
    pub blocks_written: u64,
    pub blocks_verified: u64,
    pub bad_blocks_found: u64,
    pub retry_attempts: u64,
    pub stage_timings: Vec<Duration>,
    pub throughput_samples: Vec<f64>,
}
```

### Event System

```rust
pub enum WipeEvent {
    Started,
    StageStarted,
    Progress(u64),
    MarkedBlockAsBad(u64),
    StageCompleted(Option<Rc<anyhow::Error>>),
    Retrying,
    Completed,
    Fatal(anyhow::Error),
}

pub trait WipeEventReceiver {
    fn handle(&mut self, task: &WipeTask, 
             state: &WipeState, 
             event: WipeEvent);
}
```

## Security Hardening

### Secure Memory Management

```rust
use zeroize::Zeroize;

pub struct SecureBuffer {
    data: Vec<u8>,
}

impl Drop for SecureBuffer {
    fn drop(&mut self) {
        self.data.zeroize();
    }
}
```

### Cryptographic Security

```rust
use rand_chacha::ChaCha20Rng;
use rand::SeedableRng;

pub struct SecurePatternGenerator {
    rng: ChaCha20Rng,
    seed_counter: u64,
}

impl SecurePatternGenerator {
    pub fn new() -> Self {
        let seed = Self::gather_entropy();
        Self {
            rng: ChaCha20Rng::from_seed(seed),
            seed_counter: 0,
        }
    }
    
    fn gather_entropy() -> [u8; 32] {
        // Combine multiple entropy sources
        // System random, timing jitter, hardware RNG
    }
}
```

This comprehensive backend architecture provides a robust, secure, and maintainable foundation for the Lethe data wiping utility, with clear separation of concerns and platform-specific optimizations.