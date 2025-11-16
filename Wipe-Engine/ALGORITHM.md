# Lethe Internal Working Algorithm

## Core Algorithm Overview

The Lethe secure data wiping utility implements a multi-stage sanitization algorithm with verification, error handling, and progress tracking. Here's the detailed breakdown of the internal working mechanisms:

## Main Algorithm Flow

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           MAIN ALGORITHM FLOW                                  │
└─────────────────────────────────────────────────────────────────────────────────┘

START
  │
  ▼
┌─────────────────────────────────────┐
│         1. INITIALIZATION           │
│  ┌─────────────────────────────────┐│
│  │ - Parse command line arguments  ││
│  │ - Validate input parameters     ││
│  │ - Initialize scheme repository  ││
│  │ - Setup error handling          ││
│  └─────────────────────────────────┘│
└─────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────┐
│      2. DEVICE ENUMERATION          │
│  ┌─────────────────────────────────┐│
│  │ FOR each storage device:        ││
│  │   - Query system APIs           ││
│  │   - Get device properties       ││
│  │   - Build device catalog        ││
│  │   - Create access permissions   ││
│  └─────────────────────────────────┘│
└─────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────┐
│      3. COMMAND ROUTING             │
│  ┌─────────────────────────────────┐│
│  │ IF command == "list":           ││
│  │   → DISPLAY_DEVICES()           ││
│  │ ELSE IF command == "wipe":      ││
│  │   → WIPE_OPERATION()            ││
│  │ ELSE:                           ││
│  │   → SHOW_HELP()                 ││
│  └─────────────────────────────────┘│
└─────────────────────────────────────┘
  │
  ▼
 END
```

## Wipe Operation Algorithm

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         WIPE OPERATION ALGORITHM                               │
└─────────────────────────────────────────────────────────────────────────────────┘

WIPE_OPERATION(device_id, scheme_id, parameters):
  │
  ▼
┌─────────────────────────────────────┐
│        1. TASK PREPARATION          │
│  ┌─────────────────────────────────┐│
│  │ device = find_device(device_id) ││
│  │ scheme = get_scheme(scheme_id)  ││
│  │ task = WipeTask::new(           ││
│  │   scheme, verify_mode,          ││
│  │   device.size, block_size,      ││
│  │   offset                        ││
│  │ )                               ││
│  │ state = WipeState::default()    ││
│  └─────────────────────────────────┘│
└─────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────┐
│       2. USER CONFIRMATION          │
│  ┌─────────────────────────────────┐│
│  │ display_task_summary(task)      ││
│  │ IF !auto_confirm:               ││
│  │   confirm = ask_user()          ││
│  │   IF !confirm: EXIT             ││
│  └─────────────────────────────────┘│
└─────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────┐
│        3. DEVICE ACCESS             │
│  ┌─────────────────────────────────┐│
│  │ TRY:                            ││
│  │   access = device.access()      ││
│  │ CATCH access_error:             ││
│  │   report_fatal_error()          ││
│  │   EXIT                          ││
│  └─────────────────────────────────┘│
└─────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────┐
│      4. SANITIZATION PROCESS        │
│  ┌─────────────────────────────────┐│
│  │ success = task.run(             ││
│  │   access, state, session        ││
│  │ )                               ││
│  │ IF !success: EXIT(1)            ││
│  └─────────────────────────────────┘│
└─────────────────────────────────────┘
```

## Core Sanitization Algorithm

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                      SANITIZATION PROCESS ALGORITHM                            │
└─────────────────────────────────────────────────────────────────────────────────┘

task.run(access, state, session):
  │
  ▼
┌─────────────────────────────────────────────────────────┐
│                1. STAGE ITERATION                       │
│  ┌─────────────────────────────────────────────────────┐│
│  │ FOR stage_index in 0..scheme.stages.len():         ││
│  │   state.stage = stage_index                        ││
│  │   state.at_verification = false                    ││
│  │   stage = scheme.stages[stage_index]               ││
│  │   ┌───────────────────────────────────────────────┐││
│  │   │         EXECUTE STAGE                         │││
│  │   │  success = execute_stage(                     │││
│  │   │    stage, access, state, session              │││
│  │   │  )                                            │││
│  │   │  IF !success: RETURN false                    │││
│  │   └───────────────────────────────────────────────┘││
│  │   ┌───────────────────────────────────────────────┐││
│  │   │         VERIFICATION CHECK                    │││
│  │   │  IF verify_mode == All OR                     │││
│  │   │     (verify_mode == Last AND is_last_stage):  │││
│  │   │    state.at_verification = true               │││
│  │   │    success = verify_stage(                    │││
│  │   │      stage, access, state, session            │││
│  │   │    )                                          │││
│  │   │    IF !success: RETURN false                  │││
│  │   └───────────────────────────────────────────────┘││
│  └─────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────┘
  │
  ▼
RETURN true
```

## Stage Execution Algorithm

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        STAGE EXECUTION ALGORITHM                               │
└─────────────────────────────────────────────────────────────────────────────────┘

execute_stage(stage, access, state, session):
  │
  ▼
┌─────────────────────────────────────────────────────────┐
│              1. STAGE INITIALIZATION                    │
│  ┌─────────────────────────────────────────────────────┐│
│  │ session.handle(StageStarted)                       ││
│  │ state.position = task.offset                       ││
│  │ target_size = task.total_size - task.offset        ││
│  │ generator = create_pattern_generator(stage)         ││
│  │ buffer = allocate_buffer(task.block_size)           ││
│  └─────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────────────────────────┐
│              2. BLOCK ITERATION LOOP                    │
│  ┌─────────────────────────────────────────────────────┐│
│  │ WHILE state.position < task.total_size:            ││
│  │   ┌───────────────────────────────────────────────┐││
│  │   │        CALCULATE BLOCK PARAMETERS             │││
│  │   │  remaining = total_size - position            │││
│  │   │  write_size = min(block_size, remaining)      │││
│  │   │  block_number = position / block_size         │││
│  │   └───────────────────────────────────────────────┘││
│  │   ┌───────────────────────────────────────────────┐││
│  │   │         CHECK BAD BLOCKS                      │││
│  │   │  IF bad_blocks.contains(block_number):        │││
│  │   │    position += block_size                     │││
│  │   │    CONTINUE                                   │││
│  │   └───────────────────────────────────────────────┘││
│  │   ┌───────────────────────────────────────────────┐││
│  │   │         PATTERN GENERATION                    │││
│  │   │  generator.fill_buffer(buffer, write_size)    │││
│  │   └───────────────────────────────────────────────┘││
│  │   ┌───────────────────────────────────────────────┐││
│  │   │          WRITE OPERATION                      │││
│  │   │  success = write_block_with_retry(            │││
│  │   │    access, buffer, position, write_size,      │││
│  │   │    state, session                             │││
│  │   │  )                                            │││
│  │   │  IF !success: RETURN false                    │││
│  │   └───────────────────────────────────────────────┘││
│  │   ┌───────────────────────────────────────────────┐││
│  │   │         PROGRESS UPDATE                       │││
│  │   │  state.position += write_size                 │││
│  │   │  session.handle(Progress(position))           │││
│  │   └───────────────────────────────────────────────┘││
│  └─────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────┘
  │
  ▼
RETURN true
```

## Block Write with Retry Algorithm

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                       BLOCK WRITE WITH RETRY ALGORITHM                         │
└─────────────────────────────────────────────────────────────────────────────────┘

write_block_with_retry(access, buffer, position, size, state, session):
  │
  ▼
┌─────────────────────────────────────────────────────────┐
│              RETRY LOOP                                 │
│  ┌─────────────────────────────────────────────────────┐│
│  │ attempts = 0                                       ││
│  │ WHILE attempts <= state.retries_left:              ││
│  │   ┌───────────────────────────────────────────────┐││
│  │   │         SEEK TO POSITION                      │││
│  │   │  TRY:                                         │││
│  │   │    access.seek(position)                      │││
│  │   │  CATCH seek_error:                            │││
│  │   │    attempts++                                 │││
│  │   │    IF attempts > retries: GOTO MARK_BAD       │││
│  │   │    session.handle(Retrying)                   │││
│  │   │    sleep(RETRY_BACKOFF_SECONDS)               │││
│  │   │    CONTINUE                                   │││
│  │   └───────────────────────────────────────────────┘││
│  │   ┌───────────────────────────────────────────────┐││
│  │   │         WRITE OPERATION                       │││
│  │   │  TRY:                                         │││
│  │   │    access.write(buffer[0..size])              │││
│  │   │    access.flush()                             │││
│  │   │    RETURN true  // Success!                   │││
│  │   │  CATCH write_error:                           │││
│  │   │    attempts++                                 │││
│  │   │    IF attempts > retries: GOTO MARK_BAD       │││
│  │   │    session.handle(Retrying)                   │││
│  │   │    sleep(RETRY_BACKOFF_SECONDS)               │││
│  │   │    CONTINUE                                   │││
│  │   └───────────────────────────────────────────────┘││
│  └─────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────────────────────────┐
│            MARK_BAD: BAD BLOCK HANDLING                 │
│  ┌─────────────────────────────────────────────────────┐│
│  │ block_number = position / task.block_size          ││
│  │ state.bad_blocks.mark(block_number)                ││
│  │ session.handle(MarkedBlockAsBad(block_number))     ││
│  │ RETURN true  // Continue with next block           ││
│  └─────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────┘
```

## Verification Algorithm

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         VERIFICATION ALGORITHM                                 │
└─────────────────────────────────────────────────────────────────────────────────┘

verify_stage(stage, access, state, session):
  │
  ▼
┌─────────────────────────────────────────────────────────┐
│            VERIFICATION SETUP                           │
│  ┌─────────────────────────────────────────────────────┐│
│  │ generator = create_pattern_generator(stage)         ││
│  │ read_buffer = allocate_buffer(task.block_size)      ││
│  │ expected_buffer = allocate_buffer(task.block_size)  ││
│  │ state.position = task.offset                       ││
│  └─────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────────────────────────┐
│            VERIFICATION LOOP                            │
│  ┌─────────────────────────────────────────────────────┐│
│  │ WHILE state.position < task.total_size:            ││
│  │   ┌───────────────────────────────────────────────┐││
│  │   │      CALCULATE READ PARAMETERS                │││
│  │   │  remaining = total_size - position            │││
│  │   │  read_size = min(block_size, remaining)       │││
│  │   │  block_number = position / block_size         │││
│  │   └───────────────────────────────────────────────┘││
│  │   ┌───────────────────────────────────────────────┐││
│  │   │         SKIP BAD BLOCKS                       │││
│  │   │  IF bad_blocks.contains(block_number):        │││
│  │   │    position += block_size                     │││
│  │   │    CONTINUE                                   │││
│  │   └───────────────────────────────────────────────┘││
│  │   ┌───────────────────────────────────────────────┐││
│  │   │         READ AND VERIFY                       │││
│  │   │  access.seek(position)                        │││
│  │   │  bytes_read = access.read(read_buffer)        │││
│  │   │  generator.fill_buffer(expected_buffer,       │││
│  │   │                        bytes_read)            │││
│  │   │  ┌─────────────────────────────────────────┐  │││
│  │   │  │        COMPARE BUFFERS                  │  │││
│  │   │  │  FOR i in 0..bytes_read:               │  │││
│  │   │  │    IF read_buffer[i] != expected[i]:   │  │││
│  │   │  │      report_verification_error()       │  │││
│  │   │  │      RETURN false                      │  │││
│  │   │  └─────────────────────────────────────────┘  │││
│  │   └───────────────────────────────────────────────┘││
│  │   ┌───────────────────────────────────────────────┐││
│  │   │         PROGRESS UPDATE                       │││
│  │   │  state.position += read_size                  │││
│  │   │  session.handle(Progress(position))           │││
│  │   └───────────────────────────────────────────────┘││
│  └─────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────┘
  │
  ▼
RETURN true
```

## Pattern Generation Algorithms

### Random Pattern Generation

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                      RANDOM PATTERN GENERATION                                 │
└─────────────────────────────────────────────────────────────────────────────────┘

RandomGenerator::fill_buffer(buffer, size):
  │
  ▼
┌─────────────────────────────────────────────────────────┐
│          CHACHA20 RANDOM GENERATION                     │
│  ┌─────────────────────────────────────────────────────┐│
│  │ IF !initialized:                                   ││
│  │   seed = system_entropy()                          ││
│  │   rng = ChaCha20Rng::from_seed(seed)               ││
│  │   initialized = true                               ││
│  │                                                    ││
│  │ chunks = size / 8  // 64-bit chunks                ││
│  │ FOR i in 0..chunks:                                ││
│  │   random_u64 = rng.next_u64()                      ││
│  │   buffer[i*8..(i+1)*8] = random_u64.to_bytes()    ││
│  │                                                    ││
│  │ remainder = size % 8                               ││
│  │ IF remainder > 0:                                  ││
│  │   random_bytes = rng.next_u64().to_bytes()         ││
│  │   buffer[chunks*8..] = random_bytes[0..remainder]  ││
│  └─────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────┘
```

### Constant Pattern Generation

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                      CONSTANT PATTERN GENERATION                               │
└─────────────────────────────────────────────────────────────────────────────────┘

ConstantGenerator::fill_buffer(buffer, size, pattern):
  │
  ▼
┌─────────────────────────────────────────────────────────┐
│           FAST MEMORY FILL                              │
│  ┌─────────────────────────────────────────────────────┐│
│  │ MATCH pattern:                                     ││
│  │   0x00: memset(buffer, 0x00, size)                ││
│  │   0xFF: memset(buffer, 0xFF, size)                ││
│  │   other:                                           ││
│  │     FOR i in 0..size:                             ││
│  │       buffer[i] = pattern                         ││
│  └─────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────┘
```

## Bad Block Tracking Algorithm

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        BAD BLOCK TRACKING ALGORITHM                            │
└─────────────────────────────────────────────────────────────────────────────────┘

RoaringBlockMarker Implementation:
  │
  ▼
┌─────────────────────────────────────────────────────────┐
│         ROARING BITMAP OPERATIONS                       │
│  ┌─────────────────────────────────────────────────────┐│
│  │ Internal Structure: RoaringBitmap<u32>             ││
│  │                                                    ││
│  │ mark(block_number):                                ││
│  │   bitmap.insert(block_number as u32)               ││
│  │                                                    ││
│  │ contains(block_number):                            ││
│  │   RETURN bitmap.contains(block_number as u32)      ││
│  │                                                    ││
│  │ Memory Efficiency:                                 ││
│  │   - Sparse representation for scattered bad blocks ││
│  │   - Dense representation for clustered bad blocks  ││
│  │   - Automatic optimization based on density        ││
│  └─────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────┘
```

## Progress Tracking Algorithm

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        PROGRESS TRACKING ALGORITHM                             │
└─────────────────────────────────────────────────────────────────────────────────┘

Progress Calculation:
  │
  ▼
┌─────────────────────────────────────────────────────────┐
│           PROGRESS METRICS                              │
│  ┌─────────────────────────────────────────────────────┐│
│  │ total_bytes = task.total_size - task.offset        ││
│  │ bytes_done = state.position - task.offset          ││
│  │ percentage = (bytes_done * 100) / total_bytes      ││
│  │                                                    ││
│  │ current_time = now()                               ││
│  │ elapsed = current_time - start_time                ││
│  │ speed = bytes_done / elapsed.as_seconds()          ││
│  │ eta = (total_bytes - bytes_done) / speed           ││
│  │                                                    ││
│  │ Progress Display:                                  ││
│  │   [████████████████████░░░░] 75%                   ││
│  │   Speed: 125.3 MB/s | ETA: 0:42:15                ││
│  │   Stage 2/3: Random Fill | Verification: Off      ││
│  └─────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────┘
```

## Error Recovery Strategy

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         ERROR RECOVERY STRATEGY                                │
└─────────────────────────────────────────────────────────────────────────────────┘

Error Classification and Handling:

1. TRANSIENT ERRORS (Retry with backoff):
   ┌─────────────────────────────────────┐
   │ - Temporary I/O failures            │
   │ - Device busy conditions            │
   │ - Network storage timeouts          │
   │ - Retry up to configured limit      │
   │ - Exponential backoff: 3s, 6s, 12s │
   └─────────────────────────────────────┘

2. PERSISTENT ERRORS (Mark bad block):
   ┌─────────────────────────────────────┐
   │ - Hardware sector failures          │
   │ - Permanent read/write errors       │
   │ - Add to bad block bitmap           │
   │ - Continue with next block          │
   └─────────────────────────────────────┘

3. FATAL ERRORS (Abort operation):
   ┌─────────────────────────────────────┐
   │ - Permission denied                 │
   │ - Device disconnected               │
   │ - Insufficient system resources     │
   │ - Report error and exit             │
   └─────────────────────────────────────┘
```

## Performance Optimization Techniques

### Block Size Optimization
- **Small Blocks (4KB-64KB)**: Better for error isolation, slower overall
- **Medium Blocks (1MB-4MB)**: Balanced performance and error handling
- **Large Blocks (8MB-64MB)**: Maximum throughput, less error granularity

### Memory Management
- **Buffer Reuse**: Single allocation per operation, reuse across blocks
- **Pattern Caching**: Pre-generate patterns for repeated use
- **Direct I/O**: Bypass OS page cache for security and performance

### Parallel Processing Considerations
- **Current**: Single-threaded for security and simplicity
- **Future**: Potential for parallel verification passes
- **Limitation**: Most storage devices perform better with sequential access

This comprehensive algorithm documentation provides the complete internal working mechanism of the Lethe secure data wiping utility.