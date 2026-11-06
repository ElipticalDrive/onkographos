# CPU-Driven Task Spawning

Also called: CPU-controlled GPU execution, CPU-driven thread spawning

## Overview
CPU-driven task spawning is a GPU execution model where the CPU acts as the central governor, deciding when and how GPU tasks are launched, while the GPU functions as a coprocessor that executes work requests but cannot generate new ones [S1, unofficial]. This approach centralizes decision-making logic but suffers from high CPU-GPU communication latency, making it ineffective for pipelines with fine-grain micro-dependencies or stages that dynamically compute the work for subsequent stages [S1, unofficial]. It differs from GPU-driven spawning (where the GPU generates its own work) and asynchronous submission (which hides latency but still relies on CPU-initiated commands) by strictly enforcing a top-down control hierarchy [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Volumetric data structures (e.g., textures, sparse grids) and CPU-side control logic determining task parameters (dispatch counts, buffer addresses).
*   **Output:** GPU execution of compute, rasterization, or ray-tracing tasks; completion signals returned to the CPU.

**Data Structures:**
*   **Command Buffers/Lists:** Linear memory structures containing encoded GPU commands. In modern APIs, these may be pre-allocated arrays of command headers or indirect parameter blocks.
*   **Submission Queues:** Ordered lists of commands submitted to the GPU hardware.
*   **Indirect Parameter Blocks:** GPU memory locations containing dynamic parameters (e.g., number of workgroups, vertex counts) read by the GPU at execution time.

**Numbered Steps:**
1.  **CPU Decision:** The CPU analyzes the current state of the volume rendering pipeline (e.g., visibility culling results, load balancing metrics) to determine the necessary work for the GPU.
2.  **Command Encoding:** The CPU encodes GPU commands into a command buffer. If using indirect submission, the CPU encodes a command with a pointer to GPU memory rather than immediate parameter values [S1, unofficial].
3.  **Submission:** The CPU submits the command buffer to a GPU submission queue.
4.  **Execution:** The GPU hardware fetches commands from the queue. If indirect, it reads the parameter block from GPU memory to determine execution parameters [S1, unofficial].
5.  **Synchronization:** The CPU may wait for a completion signal (fence/sync object) before proceeding, incurring round-trip latency [S1, unofficial].

**Formulas:**
*   **Round-Trip Latency ($L_{RT}$):** The time elapsed between CPU submission and CPU receipt of completion signal.
    $$ L_{RT} \approx 20\text{--}30 \, \mu s $$
    This value is measured for empty submissions and is independent of GPU type (integrated vs. dedicated) [S1, unofficial].

**Parameters:**
*   **Task Size:** Can be set to zero to cancel a submission, but the submission overhead remains [S1, unofficial].
*   **Queue Count:** Limited by hardware/API. Vulkan reports 24 queues for Nvidia, 16 for AMD, 5 for Intel, and 1 for mobile [S1, unofficial]. CUDA maps unlimited streams to 32 internal queues [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Latency Bottleneck:** The 20–30 µs round-trip latency is significant compared to sub-nanosecond CPU/GPU clocks, creating a bottleneck for fine-grained control [S1, unofficial].
*   **Indirect Submission Overhead:** If the number of pipeline passes is unknown, the CPU must issue indirect submissions for a maximum number of passes and cancel unnecessary ones, incurring overhead repeatedly [S1, unofficial].
*   **Barrier Granularity:** Barriers enforce dependencies at a coarse granularity, potentially stalling more GPU work than necessary [S1, unofficial].
*   **Queue Limitations:** The limited number of submission queues (e.g., 24 max on Nvidia) is insufficient for enforcing dependencies at the GPU core granularity (50–100 cores) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# CPU-side logic for CPU-driven task spawning
def cpu_driven_spawn(volume_data, culling_results):
    # 1. CPU decides work based on culling results
    num_workgroups = calculate_workgroups(culling_results)
    
    # 2. Encode command
    # If using indirect submission, write parameters to GPU memory first
    if use_indirect:
        gpu_param_block = allocate_gpu_memory()
        write_to_gpu_memory(gpu_param_block, num_workgroups)
        command = create_indirect_command(address=gpu_param_block)
    else:
        command = create_direct_command(workgroups=num_workgroups)
        
    # 3. Submit to queue
    submit_to_queue(command)
    
    # 4. Wait for completion (incurring latency)
    wait_for_fence()
```

## Subtopics

### [Asynchronous Submission](5531-asynchronous-submission.md)
Asynchronous submission is a model where GPU commands are placed into a queue and executed by the GPU as soon as required resources are available, but they return immediately from the point of view of the CPU. It solves the problem of high CPU-GPU communication latency inherent in synchronous control, allowing the CPU and GPU to work concurrently. Unlike synchronous submission, which blocks the CPU until the GPU completes a task, asynchronous submission removes round-trip latency and enables parallel execution of many small tasks or chaining of big tasks.

### Indirect Submission
Indirect submission is a model where the CPU emits all GPU commands, but the parameters of said commands are filled by the GPU. The command contains an address in GPU memory, which the GPU reads before execution to determine parameters [S1, unofficial]. A first pass writes to the pointer in GPU memory to decide the command's parameters, and a barrier must be placed between the two passes to ensure correct execution [S1, unofficial]. Task size can be set to zero to effectively cancel a submission, but the submission overhead is still incurred [S1, unofficial]. It is not possible to dynamically create new submissions from the GPU; if the number of required passes is not known in advance, one must issue indirect submissions for a maximum number of passes and dynamically cancel unnecessary ones, paying overhead many times [S1, unofficial]. It is an awkward hybrid model where the GPU controls most aspects but cannot initiate submissions, making it unintuitive and error-prone [S1, unofficial]. It is widely available and in use today for indirect control of task size (number of workgroups, vertices, rays), with advanced iterations including DirectX ExecuteIndirect and Vulkan VK_EXT_device_generated_commands [S1, unofficial].

### Multiple submission queues
Multiple submission queues is a solution where GPU commands are submitted to specific queues instead of directly to the GPU. Barriers only enforce dependencies between tasks of the same queue, allowing tasks in different queues to run independently [S1, unofficial]. For example, submitting task C to a different queue than task A and B lets it run independently of A or B [S1, unofficial]. Vulkan reports 24 queues for Nvidia GPUs, 16 for AMD, 5 for Intel, and usually only 1 for mobiles [S1, unofficial]. CUDA lets you have an unlimited number of streams, but behind the scenes maps them to 32 internal queues [S1, unofficial]. The restricted number of queues is not enough for micro-dependencies; modern GPUs can have 50 to 100 cores, so it is not possible to use queues to enforce dependencies at the GPU core granularity [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Submission latency is dominated by driver overhead and CPU-GPU communication, measured at 20–30 µs per round trip [S1, unofficial]. This latency is constant regardless of GPU type (integrated or dedicated) [S1, unofficial].
*   **Memory Complexity:** Requires command buffers and potentially indirect parameter blocks in GPU memory.
*   **Measured Performance:** The 20–30 µs latency is significant for fine-grained tasks, creating a bottleneck for pipelines with micro-dependencies [S1, unofficial].
*   **Scaling:** Scaling is limited by the number of submission queues (e.g., 24 on Nvidia) and the inability to enforce dependencies at the core level (50–100 cores) [S1, unofficial].
*   **Strengths:** Centralized control logic; simple programming model for coarse-grained tasks.
*   **Limitations:** High latency; awkward hybrid models for indirect submission; limited queue granularity.
*   **When to Use:** Coarse-grained volume rendering pipelines where tasks are large and dependencies are minimal.
*   **When Not to Use:** Pipelines with fine-grain micro-dependencies or stages that dynamically compute work for subsequent stages [S1, unofficial].

## Background
The CPU-driven task spawning model originated from the traditional view of GPUs as coprocessors, where the CPU is the central processing unit and the GPU executes work requests [S1, unofficial]. This model was theoretically simple but became a bottleneck due to high CPU-GPU latency [S1, unofficial]. Techniques such as asynchronous submission, indirect submission, and multiple queues were invented to partially solve these issues, but they remain imperfect solutions [S1, unofficial]. The model is widely used in modern graphics APIs (Vulkan, DirectX, CUDA) but is increasingly contrasted with GPU-driven approaches for complex, dynamic workloads [S1, unofficial].

## Sources
[S1] Real-Time Rendering of Ultra-Large and Ultra-Detailed Virtual World using Voxels, https://hal.science/tel-05803730v1, fetched 2026-10-26 (unofficial)
