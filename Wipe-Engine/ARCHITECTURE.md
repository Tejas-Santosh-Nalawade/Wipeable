# Wineable - Secure Data Wiping Tool Architecture

## System Overview

Wipeable is a cross-platform secure data wiping utility written in Rust that provides cryptographically secure deletion of storage devices. It uses industry-standard sanitization schemes and multiple overwriting passes to ensure data cannot be recovered.

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                 Wipeable ARCHITECTURE                              │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────────────────┐ │
│  │   CLI INTERFACE │    │   UI FRONTEND   │    │      ARGUMENT PARSER        │ │
│  │     (main.rs)   │◄──►│    (ui/cli.rs)  │◄──►│      (ui/args.rs)           │ │
│  └─────────────────┘    └─────────────────┘    └─────────────────────────────┘ │
│           │                       │                           │                  │
│           ▼                       ▼                           ▼                  │
│  ┌─────────────────────────────────────────────────────────────────────────────┐ │
│  │                        CONTROL LAYER                                       │ │
│  │  ┌─────────────────┐           ┌─────────────────────────────────────────┐ │ │
│  │  │   STORAGE REPO  │           │         ACTION HANDLERS                 │ │ │
│  │  │(ui/storage_repo)│           │        (actions/wipe.rs)                │ │ │
│  │  └─────────────────┘           └─────────────────────────────────────────┘ │ │
│  └─────────────────────────────────────────────────────────────────────────────┘ │
│           │                                           │                          │
│           ▼                                           ▼                          │
│  ┌─────────────────────────────────────────────────────────────────────────────┐ │
│  │                         CORE LAYER                                          │ │
│  │  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────────────┐ │ │
│  │  │   SANITIZATION  │    │     STORAGE     │    │      BAD BLOCK         │ │ │
│  │  │  SCHEMES & MEM  │    │     ACCESS      │    │      TRACKING          │ │ │
│  │  │(sanitization/*) │    │   (storage/*)   │    │   (actions/marker.rs)  │ │ │
│  │  └─────────────────┘    └─────────────────┘    └─────────────────────────┘ │ │
│  └─────────────────────────────────────────────────────────────────────────────┘ │
│           │                           │                           │               │
│           ▼                           ▼                           ▼               │
│  ┌─────────────────────────────────────────────────────────────────────────────┐ │
│  │                        PLATFORM LAYER                                      │ │
│  │  ┌─────────────────┐                       ┌─────────────────────────────┐ │ │
│  │  │    UNIX/NIX     │                       │        WINDOWS              │ │ │
│  │  │  (storage/nix)  │                       │     (storage/windows)       │ │ │
│  │  │   - Linux       │                       │     - Access Control       │ │ │
│  │  │   - macOS       │                       │     - WinAPI Integration    │ │ │
│  │  └─────────────────┘                       └─────────────────────────────┘ │ │
│  └─────────────────────────────────────────────────────────────────────────────┘ │
│                                      │                                           │
│                                      ▼                                           │
│  ┌─────────────────────────────────────────────────────────────────────────────┐ │
│  │                        HARDWARE LAYER                                      │ │
│  │             Storage Devices (HDDs, SSDs, USB, etc.)                        │ │
│  └─────────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────────┘
```

## Core Components

### 1. **Main Application (`main.rs`)**
- Entry point with CLI argument parsing using `clap`
- Command dispatcher for `list` and `wipe` operations
- Integration of all subsystems

### 2. **User Interface Layer (`ui/`)**
- **CLI Frontend (`cli.rs`)**: Interactive console interface with progress bars
- **Arguments Parser (`args.rs`)**: Command-line argument validation and parsing
- **Storage Repository (`storage_repo.rs`)**: Device enumeration and management
- **ID Shortcuts (`idshortcuts.rs`)**: User-friendly device identification

### 3. **Action Layer (`actions/`)**
- **Wipe Controller (`wipe.rs`)**: Main wiping orchestration logic
- **Block Marker (`marker.rs`)**: Bad block tracking using Roaring Bitmaps
- **Task Management**: Progress tracking and error handling

### 4. **Sanitization Engine (`sanitization/`)**
- **Scheme Repository (`mod.rs`)**: Predefined wiping schemes (DoD, GOST, etc.)
- **Stage Definition (`stage.rs`)**: Individual wiping pass configurations
- **Memory Management (`mem.rs`)**: Secure random data generation

### 5. **Storage Abstraction (`storage/`)**
- **Cross-platform Interface**: Unified API for storage access
- **Unix Implementation (`nix/`)**: Linux and macOS support
- **Windows Implementation (`windows/`)**: Windows-specific APIs

## Key Features & Technologies

### Security Features
- **Multiple Wiping Schemes**: DoD 5220.22-M, GOST R 50739-95, VSITR, etc.
- **Cryptographic Random Generation**: Uses ChaCha20 PRNG for secure data
- **Verification**: Read-back verification after each stage
- **Bad Block Handling**: Automatic tracking and skipping of damaged sectors

### Performance Optimizations
- **Configurable Block Sizes**: Optimized I/O operations
- **Memory-Mapped I/O**: Efficient data handling
- **Progress Tracking**: Real-time progress reporting
- **Retry Logic**: Automatic retry for transient errors

### Cross-Platform Support
- **Windows**: WinAPI integration with proper access control
- **Linux**: Direct device access with system permissions
- **macOS**: Core Foundation integration with disk arbitration

## Data Structures

### Core Types
```rust
pub struct WipeTask {
    pub scheme: Scheme,           // Sanitization scheme
    pub verify: Verify,           // Verification level
    pub total_size: u64,          // Device size
    pub block_size: usize,        // I/O block size
    pub offset: u64,              // Starting offset
}

pub struct WipeState {
    pub stage: usize,             // Current stage
    pub at_verification: bool,    // Verification phase
    pub position: u64,            // Current position
    pub retries_left: u32,        // Retry attempts
    pub bad_blocks: BlockMarker,  // Bad block tracking
}
```

### Sanitization Schemes
```rust
pub struct Scheme {
    pub description: String,      // Human-readable description
    pub stages: Vec<Stage>,       // Sequence of wiping stages
}

pub enum Stage {
    Zero,                         // All zeros
    One,                          // All ones
    Random,                       // Cryptographic random
    Constant(u8),                 // Fixed byte pattern
}
```

## Technology Stack

### Core Dependencies
- **Rust Language**: Memory-safe systems programming
- **anyhow**: Error handling and context
- **clap**: Command-line argument parsing
- **rand/rand_chacha**: Cryptographic random generation
- **indicatif**: Progress bar and UI components
- **prettytable**: Formatted table output

### Platform-Specific Dependencies
- **Unix**: `nix`, `sysfs-class` for system integration
- **Windows**: `winapi`, `widestring` for Windows APIs
- **Cross-platform**: `libc` for low-level operations

### Data Structures
- **roaring**: Efficient bitmap for bad block tracking
- **streaming-iterator**: Memory-efficient data processing
- **regex**: Pattern matching for device identification

## Security Considerations

### Data Sanitization Standards
1. **Single Pass**: Zero or random fill
2. **DoD 5220.22-M**: Zero → One → Random
3. **VSITR/RCMP**: 7-pass alternating pattern
4. **GOST R 50739-95**: Zero → Random

### Verification Modes
- **No Verification**: Fast but less secure
- **Last Stage Only**: Balance of speed and security
- **All Stages**: Maximum security verification

### Bad Block Handling
- Uses Roaring Bitmap for efficient storage
- Automatic retry logic with exponential backoff
- Permanent marking of consistently failing blocks

## Error Handling Strategy

### Hierarchical Error Management
1. **Storage Errors**: Hardware-level I/O failures
2. **Sanitization Errors**: Data generation and verification
3. **System Errors**: Permission and resource issues
4. **User Errors**: Invalid input and configuration

### Recovery Mechanisms
- Automatic retry with configurable attempts
- Bad block marking and skipping
- Graceful degradation for partial failures
- Detailed error reporting and logging

## Performance Characteristics

### I/O Optimization
- **Block Size Tuning**: Configurable from 512B to 64MB
- **Direct I/O**: Bypass OS caching for better security
- **Parallel Operations**: Multi-threaded where possible
- **Memory Management**: Efficient buffer reuse

### Scalability Limits
- **Maximum Blocks**: 2^32 blocks per device
- **Device Size**: Up to 4096TB with 1MB blocks
- **Concurrent Devices**: Single device at a time

This architecture provides a robust, secure, and cross-platform solution for data sanitization with proper abstraction layers and security considerations.