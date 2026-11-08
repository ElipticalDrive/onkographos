# Custom GPU Architecture for Sphere Tracing

Also called: Thread Execution Model

## Overview
A GPU architecture designed for Sphere Tracing that uses independent cores rather than multi-ALU SIMD with lockstepping, handling threading natively via a storage manager [S1, unofficial]. It differs from sibling architectures like the CUDA Raymarching Pipeline by replacing warp-based scheduling with a global double-ended queue managed by a storage manager, allowing arbitrary thread spawning and termination [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Immutable instruction memory (shared across all cores), immutable data memory (shared across all cores), and a global storage manager.
*   **Output:** Frame buffer (via `pushf` instruction).

**Data Structures:**
1.  **Storage Manager:** Implemented as a double-ended queue (deque). It holds all non-running threads. It supports coarse-grained execution priority control [S1, unofficial].
2.  **Core State:** Each core maintains:
    *   **Instruction Pointer (IP):** Points to the current instruction in the immutable instruction memory.
    *   **Mutable Registers:** 16 registers per thread.
    *   **DFU Stack:** A stack for arithmetic operations within the Data Flow Unit (DFU).
3.  **Thread State:** A thread consists of an instruction pointer and 16 mutable registers. Registers are loaded on start, saved on pause, and copied when spawning a child [S1, unofficial].

**Instruction Set:**
*   `val`: Assigns immediate values into the DFU (currently used for assigning distance function IDs) [S1, unofficial].
*   `setval`: Changes the value of a register.
*   `pack`: Reads the value of a register.
*   `pushq`: Spawns a new thread. Copies all 16 registers and pushes them onto the global queue (storage manager) for later execution by any core [S1, unofficial].
*   `drop`: Terminates the current thread and discards all registers [S1, unofficial].
*   `pushf`: Terminates the current thread and sends the pixel pointer (register `r1`) and color register (`r2`) to the frame buffer [S1, unofficial].
*   `next` and `accum`: Accelerate finding the minimum distance for a set of distance functions using a built-in accumulator [S1, unofficial].

**Numbered Steps:**
1.  **Initialization:** The storage manager is initialized as an empty double-ended queue. All cores are idle.
2.  **Dispatch:** The storage manager selects a core that is ready to receive a thread. If multiple cores are ready, one is chosen arbitrarily. A thread is popped from the storage manager and sent to the core [S1, unofficial].
3.  **Execution:** The core loads the thread's 16 registers and instruction pointer. The core executes instructions from the immutable instruction memory.
    *   Arithmetic operations use the DFU stack.
    *   Control flow is managed via `pushq`, `drop`, and `pushf`.
4.  **Thread Spawning (`pushq`):** When `pushq` is executed, the core copies the current state (16 registers + IP) and pushes it to the back/front of the storage manager (depending on priority logic not fully specified, but implied by deque usage) [S1, unofficial].
5.  **Thread Termination (`drop`):** The thread is removed from the core, registers are discarded, and the core requests a new thread from the storage manager [S1, unofficial].
6.  **Frame Buffer Output (`pushf`):** The thread is removed from the core. The values in `r1` (pixel pointer) and `r2` (color) are written to the frame buffer. The core requests a new thread [S1, unofficial].
7.  **Distance Function Acceleration:** The `next` and `accum` instructions are used to find the minimum distance among a set of distance functions, reducing the size of individual distance function code [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for the execution model
# Note: This is a conceptual representation of the hardware behavior described.

class StorageManager:
    def __init__(self):
        self.queue = []  # Double-ended queue

    def push(self, thread_data):
        # thread_data is a tuple of (ip, [r0, r1, ..., r15])
        self.queue.append(thread_data)

    def pop(self):
        if self.queue:
            return self.queue.pop(0)
        return None

class Core:
    def __init__(self, storage_manager, instruction_memory, data_memory):
        self.storage_manager = storage_manager
        self.instruction_memory = instruction_memory
        self.data_memory = data_memory
        self.ip = 0
        self.registers = [0] * 16
        self.dfu_stack = []
        self.is_busy = False

    def execute(self):
        if not self.is_busy:
            thread = self.storage_manager.pop()
            if thread:
                self.ip, self.registers = thread
                self.is_busy = True
                self.run_thread()

    def run_thread(self):
        while self.is_busy:
            instruction = self.instruction_memory[self.ip]
            self.ip += 1

            if instruction.op == 'setval':
                # Set register r to value v
                self.registers[instruction.r] = instruction.v

            elif instruction.op == 'pack':
                # Read register r to DFU stack
                self.dfu_stack.append(self.registers[instruction.r])

            elif instruction.op == 'val':
                # Assign immediate value to DFU (for DFU ID)
                self.dfu_stack.append(instruction.v)

            elif instruction.op == 'pushq':
                # Spawn new thread: copy registers and push to storage
                new_thread = (self.ip, self.registers.copy())
                self.storage_manager.push(new_thread)
                # Continue execution of current thread? 
                # Source implies spawning is an action, termination is separate.
                # Assuming current thread continues unless terminated.

            elif instruction.op == 'drop':
                # Terminate and discard
                self.is_busy = False
                self.registers = [0] * 16
                self.ip = 0
                # Request new thread
                self.execute() 

            elif instruction.op == 'pushf':
                # Terminate and send r1 (pixel), r2 (color) to frame buffer
                pixel_ptr = self.registers[1]
                color = self.registers[2]
                frame_buffer.write(pixel_ptr, color)
                self.is_busy = False
                self.registers = [0] * 16
                self.ip = 0
                # Request new thread
                self.execute()

            elif instruction.op == 'next':
                # Accelerate min distance calculation
                # Built-in accumulator logic
                pass

            elif instruction.op == 'accum':
                # Built-in accumulator logic
                pass

            else:
                # Arithmetic operations using DFU stack
                # ...

class FrameBuffer:
    def write(self, pixel_ptr, color):
        # Write color to pixel_ptr
        pass

# Initialization
storage = StorageManager()
fb = FrameBuffer()
cores = [Core(storage, instruction_mem, data_mem) for _ in range(N)]

# Initial threads are pushed to storage manager
# ...
```

**Parameters:**
*   **Registers per thread:** 16 [S1, unofficial].
*   **Memory:** Immutable instruction and data memory mirrored across all cores [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Arbitrary Core Selection:** If multiple cores are ready, one is chosen arbitrarily, which may affect performance predictability [S1, unofficial].
*   **Instruction Predictability:** The instruction set is restrictive to make instruction memory accesses predictable [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Native threading support, coarse-grained priority control via deque, predictable instruction memory access, reduced code size for distance functions via `next`/`accum` [S1, unofficial].
*   **Limitations:** Arbitrary core selection, potentially complex hardware implementation for storage manager and DFU [S1, unofficial].
*   **When to use:** When implementing a custom GPU architecture specifically optimized for Sphere Tracing with native threading requirements [S1, unofficial].
*   **When not to use:** When using standard SIMD GPUs or when warp-based scheduling is sufficient [S1, unofficial].

## Background
The architecture was described in a thesis evaluating the Sphere Tracing algorithm and a GPU designed to run it [S1, unofficial]. It was written in C*λ*aSH, a functional hardware description language based on Haskell [S1, unofficial]. The design focuses on handling threading natively with light-weight threads, using a storage manager for scheduling [S1, unofficial].

## Sources
[S1] Sphere Tracing GPU An evaluation of the Sphere Tracing algorithm and a GPU designed to run it, https://odr.chalmers.se/items/37b0e53f-fc76-4b0e-96d8-8e864573ada9/full, fetched 2026-10-29 (unofficial)
