## (DAC 2024) EnTurbo: Accelerate Confidential Serverless Computing via Parallelizing Enclave Startup Procedure

[link to paper](https://dl.acm.org/doi/10.1145/3649329.3658492)

![image-20250902142223307](/Users/huapeichun/Desktop/Paper-Reading/Privacy Computing/assets/image-20250902142223307.png)

The enclave startup procedure consumes 76%-97% of workload lifecycle time in SGXv1 and 43%-97% in SGXv2. The authors identified two key bottlenecks:

1. **Integrity Dependence**: Enclave execution must wait for completion of "Measurement, Verification" phases
2. **Serialized Measurement**: The **EEXTEND** primitive can only measure 512-bit chunks sequentially, accounting for ~85.1% of startup time

They thus propose:

1. Decouple Startup Procedure

   - Eliminates integrity dependence by allowing enclave execution parallel to measurement/verification

   - Uses **Host thread** for copying and execution, **Worker threads** for measurement/verification

   ![image-20250902142349796](/Users/huapeichun/Desktop/Paper-Reading/Privacy Computing/assets/image-20250902142349796.png)

2. Parallel Measurement

   - Replaces serial measurement with multi-threaded measurement

   - Multiple Worker threads measure different page blocks simultaneously

   ![image-20250902142425685](/Users/huapeichun/Desktop/Paper-Reading/Privacy Computing/assets/image-20250902142425685.png)

### Problem 1: Conflicts Between Threads

**Challenge**: Host and Worker threads may interfere during parallel execution. (1) **Input Conflict**: The Host thread modifies enclave pages at Copying and Execution phase, disrupting the measurement input of the Worker thread. (2) **Update conflict**: Both Worker and Host threads update the measurement value to ensure the integrity of the startup procedure.

**Solution**:

- **Input Conflict**: Deploy serial "Copy-Measure-Execute" page access sequence using VALID and WRLock flags in Enclave Page Cache Map (EPCM) entries. This ensures: (R1) All pages must be copied before being measured. (R2) Writable pages cannot be modified at Execution phase before they are measured by Worker threads.

  ![image-20250902142706007](/Users/huapeichun/Desktop/Paper-Reading/Privacy Computing/assets/image-20250902142706007.png)

- **Update Conflict**: Implement Duplicate-Merge mechanism using **Shadow Measurement Cache (SMC)** to store intermediate hash results separately

  ![image-20250902142757881](/Users/huapeichun/Desktop/Paper-Reading/Privacy Computing/assets/image-20250902142757881.png)

### Problem 2: Security Before Verification

**Challenge**: Enclave may leak secrets before integrity verification completes

**Solution**:

- **Memory Access Protection**: Use **Intel Memory Protection Key (MPK)** to block write-out operations

- **Event Blocking**: Prevent information leakage through **Asynchronous Enclave Exit (AEX)** and **OCALL** events using HLT instruction

  ![image-20250902142936044](/Users/huapeichun/Desktop/Paper-Reading/Privacy Computing/assets/image-20250902142936044.png)

### Problem 3: Measurement Security

**Challenge**: Parallel measurement changes the original SHA-256 hash sequence

**Solution**:

- Mathematically prove parallel measurement maintains equivalent security to original SGX
- Use **HASH_DOM** flags to ensure correct measurement ordering across Worker threads

### Evaluation Results

Testing on SGX simulation mode with confidential serverless workloads:

- Optimal configuration: 8 Worker threads based on empirical analysis. When the number of Work threads exceeds 8, the cost performance of each thread decelerates.

  ![image-20250902143557387](/Users/huapeichun/Desktop/Paper-Reading/Privacy Computing/assets/image-20250902143557387.png)

- Performance improvement: 1.42x-6.48x speedup on SGXv1, 1.33x-3.76x on SGXv2

- Memory overhead: Only 4KB additional memory for SMC

- Hardware overhead: Negligible (uses reserved bits in existing structures)

![image-20250902143418116](/Users/huapeichun/Desktop/Paper-Reading/Privacy Computing/assets/image-20250902143418116.png)![image-20250902143435810](/Users/huapeichun/Desktop/Paper-Reading/Privacy Computing/assets/image-20250902143435810.png)

### Thoughts

What Constitutes a "Function" in FaaS?

- **Unit of execution**: A single entry point (e.g., `handler(event, context)`) that processes one request
- Lifetime: Typically milliseconds to seconds
  - Image thumbnail generation (100-500ms)
  - API gateway request handler (50-200ms)
  - Database query processor (10-100ms)
  - ML inference (100ms-2s)
- Each function invocation is isolated. The enclave lifecycle matches the function lifecycle. Cold start happens for each invocation (worst case). This brings:
  - Fault Isolation;
  - Automatic Scaling;
  - Pay-per-use.

**Q: Why Not Reuse Enclaves?** 🤔 

```
Traditional: Create → Load → Execute → Destroy
Optimized:   Create → Load → Execute → Reset → Execute → ... → Destroy
                              ↑_______Loop_______↑
```

A. Library OS Approach (Gramine, Occlum)

- Keep enclave alive across invocations
- Reset application state between functions

B. Snapshot/Fork (Penglai, PIE)

- Create template enclave -> Fork for each invocation
- Still has startup overhead but amortizes binary loading

Challenges:

- Memory must be cleared (including stack, heap, registers)
- File descriptors must be reset
- Random number generators must be reseeded
- TLS sessions must be terminated
- ... (Numerus others)

**Q: For SEV, what is different?** 🤔

```
┌─────────────────────────────────────┐
│         SEV Service VM              │
├─────────────────────────────────────┤
│   Function Scheduler/Multiplexer    │
├──────┬──────┬──────┬──────┬────────┤
│ Slot1│ Slot2│ Slot3│ Slot4│  ...   │
│ F(A) │ F(B) │ F(A) │ F(C) │        │
├──────┴──────┴──────┴──────┴────────┤
│    Secure State Management Layer    │
├─────────────────────────────────────┤
│    SEV Hardware Encryption          │
└─────────────────────────────────────┘
```

SEV works at the VM-level instead of process-level, and is therefore more coarse-grained.

- Context and memory isolation will be done at software level instead of guaranteed by construction. Scheduler is designed to map different functions to a "sandboxed" region with guard pages.
- Between invocations, state verification with crypto foundation is needed.
  - Merkle trees? 🤔
    - Clean state has known merkle root
    - SHA is also fast on AMD
  - Pre-allocate fixed execution slots, rotate functions through them? 🤔
    - AMD CPUs have fast memory zeroing instructions
    - Slots can be sized to fit in L3 cache -> fast for small scale operations
    - Function in Slot A tries to access Slot B's memory -> MPK enforcement (I don't know if MPK exists in AMD actually)
  - Timing side-channels? 🤔