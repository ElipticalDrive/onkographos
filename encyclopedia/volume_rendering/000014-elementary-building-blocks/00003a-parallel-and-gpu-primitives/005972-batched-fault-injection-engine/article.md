# Batched Fault Injection Engine

Also called: throughput-bound parallel workload

## Overview
The Batched Fault Injection Engine is a parallel workload that batches multiple corrupted variants of a scene into a single rasterizer call to saturate device throughput, removing latency bounds associated with host control between renders [S1, unofficial]. It differs from standard fault injection by treating the injection campaign itself as a data-parallel rendering task rather than a sequence of latency-bound host-driven calls, allowing it to sustain near-maximum hardware utilization [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A scene representation (e.g., 3D Gaussian Splatting primitives) and a set of bit-flip patterns.
- **Output:** A batch of rendered images, each corresponding to a specific corrupted variant of the scene.

**Data Structures:**
- **Batched Scene Data:** The scene primitives are duplicated $K$ times (where $K$ is the batch size) to hold independent corruption states.
- **Bit-Flip Patterns:** A vectorized structure defining which bits to flip in each variant.

**Steps:**
1. **Prepare Batch:** Duplicate the scene data $K$ times to create $K$ independent variants.
2. **Apply Corruption:** Apply bit flips to the batch using a vectorized scatter operation [S1, unofficial]. This operation targets specific memory locations corresponding to the primitive attributes for each variant in the batch.
3. **Rasterize:** Issue a single rasterizer call processing all $K$ variants simultaneously [S1, unofficial].
4. **Collect Results:** Read back the rendered outputs for each variant.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def batched_fault_injection(scene_primitives, bit_flip_patterns, batch_size):
    # Duplicate scene primitives for each variant in the batch
    batched_primitives = repeat(scene_primitives, batch_size)
    
    # Apply bit flips to the batch using a vectorized scatter operation
    # bit_flip_patterns is a structured array defining flips for each variant
    corrupted_primitives = vectorized_scatter(batched_primitives, bit_flip_patterns)
    
    # Render all variants in a single call
    # The rasterizer processes the batched primitives in parallel
    rendered_images = rasterizer_call(corrupted_primitives)
    
    return rendered_images
```

**Parameters:**
- **Batch Size ($K$):** The number of corrupted variants processed per call. Typical values saturate the device (e.g., 32 variants per launch [S1, unofficial]).
- **Bit-Flip Patterns:** Defined by the fault injection campaign (e.g., single-bit upsets [S1, unofficial]).

**Edge Cases and Pitfalls:**
- **Throughput Saturation:** The method is designed to saturate device throughput; if the batch size is too small, the device remains underutilized due to host control latency [S1, unofficial].
- **Memory Bandwidth:** High memory bandwidth is required to handle the duplicated scene data and bit-flip operations efficiently [S1, unofficial].
- **Hardware Specificity:** Throughput varies by hardware (e.g., Blackwell vs. L40S), but bit-level criticality remains consistent across architectures with identical floating-point layouts [S1, unofficial].

## Subtopics

### [Single-Event Upset (SEU) Fault Injection in 3DGS](17923-single-event-upset-seu-fault-injection-in-3dgs.md)
Simulates transient bit flips in GPU memory to measure rendering corruption in 3D Gaussian Splatting. It differs from neural network weight perturbation because 3DGS primitives are spatially localized and redundant, whereas network weights are shared globally; this method quantifies whether such redundancy provides robustness or concentrates risk into rare catastrophic failures.

## Operating characteristics
- **Time Complexity:** The method is throughput-bound, not latency-bound. It processes $K$ variants in a single call, reducing overhead per variant to near zero [S1, unofficial].
- **Memory Complexity:** Requires $K$ times the memory for scene primitives, where $K$ is the batch size [S1, unofficial].
- **Performance:**
    - Sustains 2,515 single-bit injections per second on Blackwell hardware [S1, unofficial].
    - Processes 4.3 million primitive instances per launch [S1, unofficial].
    - Reaches 6,081 single-bit injections per second on 4 L40S accelerators (4.00x speedup) with 100% parallel efficiency [S1, unofficial].
    - Saturates device at 99% utilization and 386 W [S1, unofficial].
- **Strengths:**
    - Eliminates host control latency between renders [S1, unofficial].
    - Embarrassingly parallel across devices (no communication needed) [S1, unofficial].
    - Achieves near-maximum hardware utilization [S1, unofficial].
- **Limitations:**
    - Requires significant memory for batched scene data [S1, unofficial].
    - Throughput is hardware-dependent (e.g., 1,522 injections/sec on L40S vs. 2,515 on Blackwell) [S1, unofficial].
- **When to Use:** When the goal is to measure bit-level criticality or fault tolerance at scale, and hardware throughput is the bottleneck rather than host control latency [S1, unofficial].
- **When Not to Use:** When the number of variants is small, or when host control latency is negligible compared to rendering time [S1, unofficial].

## Background
The Batched Fault Injection Engine was introduced in the context of analyzing Single-Event Upsets (SEUs) in 3D Gaussian Splatting rendering [S1, unofficial]. It was developed to address the latency-bound nature of traditional fault injection campaigns, where each corrupted render is small relative to the host control between renders [S1, unofficial]. The method was validated on Blackwell and Ada-generation (L40S) hardware, demonstrating that bit-level criticality is a property of the representation and floating-point layout rather than specific architecture [S1, unofficial].

## Sources
[S1] Single-Event Upsets in 3D Gaussian Splatting Rendering: Bit-Level Criticality, Spatial Extent, and a Parallel Support Guard, https://doi.org/10.48550/arxiv.2606.21791, fetched 2026-10-26 (unofficial)
