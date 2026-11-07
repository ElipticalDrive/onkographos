# Double Buffer Shared Warp Marching

Also called: DBShared, Double buffer approach

## Overview
Double Buffer Shared Warp Marching (DBShared) is a warp-level reduction technique for volume rendering that mitigates the thread inactivity inherent in previous shared-memory approaches like SBShared [S1]. By using two buffers per thread and delaying color blending until all threads in the warp have loaded samples, it keeps the entire warp active during reduction steps, improving execution efficiency at the cost of higher load-compute ratio and increased data movement [S1]. It builds on [Double Buffer Shuffling](5523-double-buffer-shuffling.md) by applying the double-buffer concept to shared memory allocation for ray marching [S1].

## How it works
**Inputs:**
- Ray marching parameters (origin, direction, step size, number of steps).
- Volume texture data.
- Transfer function for color and opacity derivation.

**Outputs:**
- Final accumulated color and opacity for the ray.

**Data Structures:**
Each thread in the warp maintains two buffers allocated from GPU shared memory:
1.  **Buffer 1 (Accumulator):** Stores the result of the previous reduction step. Initialized to zero at the start.
2.  **Buffer 2 (Input):** Stores the newly sampled color/opacity data for the current cycle.

Each buffer holds one operand for blending. In the standard implementation described, these are `float4` values (RGBA) [S1].

**Algorithm Steps:**

1.  **Initialization (Cycle 1, Step 1):**
    All threads in the warp initialize their Buffer 1 to zero $(0, 0, 0, 0)$ [S1].

2.  **Sample Loading (Cycle 1, Step 2):**
    Threads take samples along the ray, derive color and opacity via the transfer function, and load these values into Buffer 2 [S1].

3.  **Reduction Step 1 (Cycle 1, Step 3):**
    All threads perform a one-step reduction (blending) using data from Buffer 1 and Buffer 2. The result is written back to Buffer 1. This step effectively halves the number of active data units but ensures all threads participate [S1].

4.  **Sample Loading (Cycle 2, Step 1):**
    Before the next reduction, the warp loads another set of samples into Buffer 2 for all threads [S1].

5.  **Reduction Step 2 (Cycle 2, Step 2):**
    All threads perform another one-step reduction using the updated Buffer 1 and Buffer 2, writing results back to Buffer 1 [S1].

6.  **Subsequent Cycles:**
    Cycles from the third onwards repeat the pattern of Cycle 2:
    -   Load samples into Buffer 2.
    -   Perform one-step reduction into Buffer 1.
    
    This continues until the ray is terminated or all samples are processed. Thread inactivity only occurs in the final cycle if the number of marching steps is not a multiple of the warp size, as no new data is loaded for idle threads [S1].

**Early Ray Termination Handling:**
DBShared cannot derive the exact integrated color at each cycle due to delayed blending. To handle early ray termination, the accumulated opacity must be derived at each warp cycle [S1].
-   After the one-step reduction in each cycle, the opacity components are transferred from Buffer 1 to Buffer 2.
-   A complete reduction is performed on the opacity values in Buffer 2.
-   The result in Buffer 2 determines if the ray should terminate. If terminated, the process stops; otherwise, it continues [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def db_shared_warp_march(ray_origin, ray_direction, volume_data, transfer_function, num_steps, warp_size):
    # Shared memory allocation for the warp
    # Each thread has two buffers: buf1 (accumulator) and buf2 (input)
    # buf1 and buf2 are float4 (RGBA)
    
    # Thread-local variables
    buf1 = (0.0, 0.0, 0.0, 0.0) # R, G, B, A
    buf2 = (0.0, 0.0, 0.0, 0.0)
    
    # Cycle 1, Step 1: Initialize buf1 to zero
    buf1 = (0.0, 0.0, 0.0, 0.0)
    
    # Cycle 1, Step 2: Load first set of samples into buf2
    # Assuming thread index determines sample offset within the warp's batch
    sample_pos = ray_origin + ray_direction * (thread_id_in_warp * step_size)
    color, opacity = transfer_function(volume_data, sample_pos)
    buf2 = (color.r, color.g, color.b, opacity)
    
    # Cycle 1, Step 3: One-step reduction into buf1
    # Reduction blends buf1 and buf2. Standard front-to-back blending:
    # C_out = C_in + C_new * (1 - A_in)
    # A_out = A_in + A_new * (1 - A_in)
    buf1 = blend(buf1, buf2)
    
    # Determine if ray should terminate based on accumulated opacity
    # Note: DBShared requires opacity reduction at each cycle for termination check
    opacity_buf2 = (0.0, 0.0, 0.0, buf1.a) # Transfer opacity to buf2
    opacity_buf2 = reduce_opacity(opacity_buf2) # Complete reduction on opacity in buf2
    if opacity_buf2.a >= TERMINATION_THRESHOLD:
        return buf1 # Return current accumulated color
    
    # Subsequent Cycles
    for cycle in range(2, ceil(num_steps / warp_size) + 1):
        # Cycle k, Step 1: Load samples into buf2
        sample_pos = ray_origin + ray_direction * ((thread_id_in_warp + (cycle - 1) * warp_size) * step_size)
        
        # Handle boundary: if sample_pos is out of bounds, use zero color/opacity
        if is_out_of_bounds(sample_pos):
            color = (0.0, 0.0, 0.0)
            opacity = 0.0
        else:
            color, opacity = transfer_function(volume_data, sample_pos)
        
        buf2 = (color.r, color.g, color.b, opacity)
        
        # Cycle k, Step 2: One-step reduction into buf1
        buf1 = blend(buf1, buf2)
        
        # Check for early ray termination
        # Transfer opacity from buf1 to buf2 for reduction
        opacity_buf2 = (0.0, 0.0, 0.0, buf1.a)
        opacity_buf2 = reduce_opacity(opacity_buf2)
        
        if opacity_buf2.a >= TERMINATION_THRESHOLD:
            return buf1
            
    return buf1

def blend(buf1, buf2):
    # Front-to-back blending
    # buf1: (R1, G1, B1, A1)
    # buf2: (R2, G2, B2, A2)
    a1, a2 = buf1[3], buf2[3]
    r1, g1, b1 = buf1[0], buf1[1], buf1[2]
    r2, g2, b2 = buf2[0], buf2[1], buf2[2]
    
    # New alpha
    a_new = a1 + a2 * (1.0 - a1)
    
    # New color (weighted average)
    if a_new > 0.0:
        r_new = (r1 * a1 + r2 * a2 * (1.0 - a1)) / a_new
        g_new = (g1 * a1 + g2 * a2 * (1.0 - a1)) / a_new
        b_new = (b1 * a1 + b2 * a2 * (1.0 - a1)) / a_new
    else:
        r_new, g_new, b_new = 0.0, 0.0, 0.0
        
    return (r_new, g_new, b_new, a_new)

def reduce_opacity(buf):
    # Warp-level reduction for opacity only
    # Uses warp shuffling to sum/blend opacity across threads
    # Returns a single opacity value distributed to all threads
    # Implementation details depend on specific warp shuffle primitives
    # [G] Standard warp shuffle reduction logic would be applied here
    pass
```

**Parameters:**
-   **Buffer Allocation:** Two buffers per thread, allocated from GPU shared memory [S1].
-   **Warp Size:** Typically 32 threads, but the algorithm is described for a generic warp size (e.g., 8 in the example) [S1].

**Edge Cases and Pitfalls:**
-   **Early Ray Termination:** DBShared cannot derive the exact integrated color at each cycle. It relies on accumulated opacity for termination, which requires an additional reduction step for opacity per cycle. This may result in extra texture fetches (up to $3 \times 8$ extra operations for a warp of 8) if the ray terminates early [S1].
-   **Last Cycle Inactivity:** If the number of marching steps is not a multiple of the warp size, some threads will be idle in the last cycle because no new data is loaded. This inactivity is unavoidable but becomes a smaller percentage of total time as the number of steps increases [S1].
-   **Data Movement:** DBShared blends 8 floats (opacity) per cycle compared to SBShared blending 8 float4 (color), involving more data movements [S1].

## Operating characteristics
-   **Time Complexity:** The number of active threads remains constant (entire warp) during reduction steps, unlike SBShared where active threads halve after each step [S1]. This improves warp execution efficiency [S1].
-   **Memory Complexity:** Requires two buffers per thread in shared memory [S1].
-   **Performance:** DBShared has a higher load-compute ratio compared to SBShared (1 load, 1 step reduction vs. 1 load, 3 steps reduction) [S1]. However, it achieves better warp execution efficiency because the entire warp is active during reductions [S1].
-   **Strengths:** Mitigates thread inactivity; better warp utilization; suitable for large numbers of marching steps where last-cycle inactivity is negligible [S1].
-   **Limitations:** Higher data movement (blending opacity separately); potential for extra texture fetches due to opacity-based termination checks; increased shared memory usage per thread [S1].
-   **When to Use:** When warp execution efficiency is critical and the number of marching steps is large [S1].
-   **When Not to Use:** When memory bandwidth is severely constrained or the number of steps is small (making last-cycle inactivity significant) [S1].

## Background
DBShared was proposed as a hardware-oriented optimization for texture-based volume rendering on GPU [S1]. It addresses the drawback of the SBShared approach, where the number of active threads reduces by half after each reduction step [S1]. The method was introduced in a 2017 paper by [S1], leveraging efficient warp shuffling operations available since the Fermi architecture to implement the reduction within warps [S1].

## Sources
[S1] A cache-friendly sampling strategy for texture-based volume rendering on GPU, https://doi.org/10.1016/j.visinf.2017.08.001, fetched 2026-10-26
