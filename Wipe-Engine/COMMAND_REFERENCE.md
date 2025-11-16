# Lethe Command Reference & Usage Guide

## Overview

Lethe is a secure, cross-platform data wiping utility that uses multiple sanitization schemes to permanently delete data from storage devices. This guide covers all available commands, options, and usage patterns.

## Basic Command Structure

```bash
lethe <command> [options] [arguments]
```

## Available Commands

### 1. Device Listing Command

#### `list` - List Available Storage Devices

Lists all available storage devices that can be wiped.

**Syntax:**
```bash
lethe list
```

**Example Output:**
```
Device ID                    Short ID  Size      Type        Label    Mount Point
/dev/sda                    sda       931.5 GB  Fixed       MyDisk   /
├─ /dev/sda1                sda1      512.0 MB  Partition   EFI      /boot/efi
├─ /dev/sda2                sda2      930.0 GB  Partition   root     /
/dev/sdb                    sdb       14.9 GB   Removable   USB      /media/usb
```

**Features:**
- Shows hierarchical device structure (parent devices and partitions)
- Displays device size in human-readable format
- Shows device type (Fixed, Removable, Partition, etc.)
- Includes mount points and labels where available
- Generates short IDs for easier reference

---

### 2. Wipe Command

#### `wipe` - Securely Wipe Storage Device

Performs secure data sanitization on the specified storage device.

**Syntax:**
```bash
lethe wipe <device_id> [options]
```

**Required Arguments:**
- `<device_id>`: Storage device identifier (from `list` command)

**Available Options:**

#### `-s, --scheme <scheme>` 
Selects the data sanitization scheme to use.

**Available Schemes:**
- `zero` - Single zeroes fill
- `random` - Single random fill  
- `random2x` - Double random fill (default)
- `badblocks` - Inspired by badblocks tool -w action
- `gost` - GOST R 50739-95 (fake implementation)
- `dod` - DoD 5220.22-M / CSEC ITSG-06 / NAVSO P-5239-26
- `vsitr` - VSITR / RCMP TSSIT OPS-II

**Default:** `random2x`

**Examples:**
```bash
# Use default scheme (random2x)
lethe wipe /dev/sdb

# Use DoD standard
lethe wipe /dev/sdb --scheme dod

# Use single random pass
lethe wipe sdb -s random
```

#### `-v, --verify <mode>`
Sets verification mode for checking written data.

**Available Modes:**
- `no` - No verification (fastest)
- `last` - Verify only the last stage (default)
- `all` - Verify after each stage (most secure)

**Default:** `last`

**Examples:**
```bash
# No verification for speed
lethe wipe /dev/sdb --verify no

# Verify all stages for maximum security
lethe wipe /dev/sdb -v all
```

#### `-b, --blocksize <size>`
Sets the block size for I/O operations.

**Supported Formats:**
- Bytes: `1024`, `4096`
- Kilobytes: `4k`, `64k`  
- Megabytes: `1m`, `4m`, `64m`

**Range:** 512 bytes to 64 megabytes
**Default:** `1m`

**Examples:**
```bash
# Use 4MB blocks for better performance
lethe wipe /dev/sdb --blocksize 4m

# Use 64KB blocks for better error handling
lethe wipe /dev/sdb -b 64k
```

#### `-o, --offset <bytes>`
Sets the starting offset for wiping operation.

**Supported Formats:**
- Bytes: `1024`, `4096`
- Kilobytes: `4k`, `1024k`
- Megabytes: `1m`, `100m`
- Gigabytes: `1g`, `2g`

**Default:** `0`

**Examples:**
```bash
# Start wiping from 1GB offset
lethe wipe /dev/sdb --offset 1g

# Skip first 100MB
lethe wipe /dev/sdb -o 100m
```

#### `-r, --retries <number>`
Sets maximum number of retry attempts for failed operations.

**Range:** 0 to 255
**Default:** `8`

**Examples:**
```bash
# More aggressive retry for unreliable devices
lethe wipe /dev/sdb --retries 16

# No retries for fast operation
lethe wipe /dev/sdb -r 0
```

#### `-y, --yes`
Automatically confirm the operation without user prompt.

**Examples:**
```bash
# Auto-confirm for scripts
lethe wipe /dev/sdb --yes

# Combine with other options
lethe wipe /dev/sdb -s dod -v all -y
```

## Complete Command Examples

### Basic Usage Examples

```bash
# List all devices
lethe list

# Wipe USB drive with default settings
lethe wipe /dev/sdb

# Quick wipe with zero fill, no verification
lethe wipe /dev/sdb --scheme zero --verify no --yes

# Secure DoD wipe with full verification
lethe wipe /dev/sdb --scheme dod --verify all --blocksize 1m

# High-performance wipe with large blocks
lethe wipe /dev/sdb --scheme random2x --blocksize 8m --retries 16
```

### Advanced Usage Examples

```bash
# Wipe specific partition only
lethe wipe /dev/sdb1 --scheme gost

# Wipe with custom offset (skip first 100MB)
lethe wipe /dev/sdb --offset 100m --scheme random

# Maximum security wipe
lethe wipe /dev/sdb --scheme vsitr --verify all --blocksize 1m --retries 32

# Fast wipe for testing
lethe wipe /dev/sdb --scheme zero --verify no --blocksize 64m --yes

# Scripted wipe with error handling
lethe wipe /dev/sdb --scheme dod --verify last --retries 5 --yes
```

## Sanitization Schemes Detailed

### Standard Schemes

#### `zero` - Single Zero Pass
- **Description:** Single pass writing all zeros
- **Passes:** 1
- **Pattern:** 0x00
- **Use Case:** Quick wipe, basic data hiding

#### `random` - Single Random Pass  
- **Description:** Single pass with cryptographic random data
- **Passes:** 1
- **Pattern:** ChaCha20 random
- **Use Case:** Good security with minimal time

#### `random2x` - Double Random Pass (Default)
- **Description:** Two passes with different random data
- **Passes:** 2  
- **Pattern:** ChaCha20 random (2 different seeds)
- **Use Case:** Balanced security and performance

#### `badblocks` - Badblocks Compatible
- **Description:** Compatible with Linux badblocks -w
- **Passes:** 4
- **Patterns:** 0xAA → 0x55 → 0xFF → 0x00
- **Use Case:** Hardware testing and data erasure

### Government/Military Standards

#### `dod` - DoD 5220.22-M
- **Description:** US Department of Defense standard
- **Passes:** 3
- **Patterns:** 0x00 → 0xFF → Random
- **Use Case:** Government/military data sanitization

#### `gost` - GOST R 50739-95  
- **Description:** Russian Federation standard (simplified)
- **Passes:** 2
- **Patterns:** 0x00 → Random
- **Use Case:** Russian compliance requirements

#### `vsitr` - VSITR/RCMP TSSIT OPS-II
- **Description:** Canadian standard, most thorough
- **Passes:** 7
- **Patterns:** 0x00 → 0xFF → 0x00 → 0xFF → 0x00 → 0xFF → Random  
- **Use Case:** Maximum security, classified data

## Performance Considerations

### Block Size Selection

| Block Size | Performance | Error Handling | Memory Usage |
|------------|-------------|----------------|--------------|
| 4KB-64KB   | Slower      | Excellent      | Low          |
| 1MB-4MB    | Good        | Good           | Medium       |
| 8MB-64MB   | Fastest     | Basic          | High         |

### Device Type Recommendations

#### HDD (Traditional Hard Drives)
```bash
# Optimal settings for HDDs
lethe wipe /dev/sda --scheme dod --blocksize 4m --verify last
```

#### SSD (Solid State Drives)  
```bash
# Multiple random passes for SSDs
lethe wipe /dev/sda --scheme random2x --blocksize 1m --verify all
```

#### USB Flash Drives
```bash
# Conservative settings for USB devices
lethe wipe /dev/sdb --scheme badblocks --blocksize 1m --retries 16
```

#### Network Storage
```bash
# Higher retry count for network devices
lethe wipe /dev/mapper/iscsi --scheme random --blocksize 512k --retries 32
```

## Security Recommendations

### For Different Security Levels

#### **Low Security (Personal Data)**
```bash
lethe wipe /dev/sdb --scheme random --verify no
```

#### **Medium Security (Business Data)**  
```bash
lethe wipe /dev/sdb --scheme dod --verify last --blocksize 1m
```

#### **High Security (Confidential Data)**
```bash
lethe wipe /dev/sdb --scheme vsitr --verify all --blocksize 1m --retries 16
```

#### **Maximum Security (Classified Data)**
```bash
# Multiple sequential wipes with different schemes
lethe wipe /dev/sdb --scheme dod --verify all --yes
lethe wipe /dev/sdb --scheme vsitr --verify all --yes  
lethe wipe /dev/sdb --scheme random2x --verify all --yes
```

## Error Handling

### Common Error Scenarios

#### Permission Errors
```bash
# Run with appropriate privileges
sudo lethe wipe /dev/sdb

# On Windows, run as Administrator
lethe.exe wipe \\.\PhysicalDrive1
```

#### Device Busy Errors
```bash
# Unmount device first
umount /dev/sdb1
lethe wipe /dev/sdb
```

#### Bad Block Handling
- Lethe automatically detects and skips bad blocks
- Bad blocks are marked and avoided in subsequent passes
- Use higher retry counts for devices with many bad blocks

### Progress Monitoring

During operation, Lethe displays:
- Progress bar with percentage complete
- Current stage and operation
- Transfer speed (MB/s)
- Estimated time to completion
- Bad block count (if any)

**Example Progress Display:**
```
Wiping /dev/sdb (14.9 GB) with DoD 5220.22-M scheme:
Stage 2/3: Fill with 0xFF
[████████████████████░░░░] 75% | 125.3 MB/s | ETA: 0:42:15
Bad blocks: 12 | Retries: 3
```

## Script Integration

### Bash Script Example
```bash
#!/bin/bash

DEVICE="/dev/sdb"
SCHEME="dod"

# Check if device exists
if lethe list | grep -q "$DEVICE"; then
    echo "Wiping $DEVICE with $SCHEME scheme..."
    lethe wipe "$DEVICE" --scheme "$SCHEME" --yes
    if [ $? -eq 0 ]; then
        echo "Wipe completed successfully"
    else
        echo "Wipe failed" >&2
        exit 1
    fi
else
    echo "Device $DEVICE not found" >&2
    exit 1
fi
```

### PowerShell Script Example
```powershell
$Device = "\\.\PhysicalDrive1"
$Scheme = "dod"

# Check device exists
$DeviceList = & lethe list
if ($DeviceList -match [regex]::Escape($Device)) {
    Write-Host "Wiping $Device with $Scheme scheme..."
    & lethe wipe $Device --scheme $Scheme --yes
    if ($LASTEXITCODE -eq 0) {
        Write-Host "Wipe completed successfully"
    } else {
        Write-Error "Wipe failed"
        exit 1
    }
} else {
    Write-Error "Device $Device not found"
    exit 1
}
```

## Platform-Specific Notes

### Linux
- Requires root privileges for direct device access
- Device names: `/dev/sda`, `/dev/nvme0n1`, etc.
- WSL is not supported

### macOS  
- Requires admin privileges
- Device names: `/dev/disk0`, `/dev/disk1`, etc.
- May need to disable System Integrity Protection for some operations

### Windows
- Requires Administrator privileges  
- Device names: `\\.\PhysicalDrive0`, `\\.\PhysicalDrive1`, etc.
- Short IDs: `pd0`, `pd1`, etc.

## Troubleshooting

### Common Issues and Solutions

#### "Permission denied" errors
- **Linux/macOS:** Run with `sudo`
- **Windows:** Run as Administrator

#### "Device busy" errors  
- Unmount/eject the device before wiping
- Close any applications using the device

#### Very slow performance
- Try larger block sizes (`--blocksize 8m`)
- Disable verification (`--verify no`)
- Check if device is failing hardware-wise

#### Operation stops with errors
- Increase retry count (`--retries 32`)  
- Check device health with SMART tools
- Try smaller block sizes for better error isolation

#### Out of memory errors
- Reduce block size (`--blocksize 1m` or smaller)
- Close other memory-intensive applications

This command reference provides comprehensive coverage of all Lethe functionality and usage patterns for secure data wiping operations.