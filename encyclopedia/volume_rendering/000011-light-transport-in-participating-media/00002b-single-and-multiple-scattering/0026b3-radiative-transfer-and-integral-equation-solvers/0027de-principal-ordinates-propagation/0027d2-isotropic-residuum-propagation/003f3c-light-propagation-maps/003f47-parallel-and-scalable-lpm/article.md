# Parallel and Scalable LPM

Also called: GPU LPM with propagation and collecting steps

## Overview
Parallel and Scalable LPM is a GPU-optimized implementation of Light Propagation Maps (LPM) that solves the synchronization bottlenecks inherent in naive parallel ray tracing within participating media. Unlike standard parallel approaches that assign one thread per ray and rely on expensive atomic operations or synchronization barriers to handle concurrent writes to the same voxel, this method decouples the process into a **propagation step** and a **collecting step**. By grouping rays by propagation direction and using temporary buffers, it guarantees that each voxel is traversed by only one ray at a time within a specific direction block, thereby eliminating race conditions and leveraging data locality for higher efficiency [S1].

## How it works
The method implements the LPM algorithm on a GPU by splitting the iterative light propagation into two distinct kernel launches per iteration: a parallel propagation phase and a sequential collecting phase.

### Inputs and Outputs
*   **Inputs:**
    *   Volume grid with spatial resolution $V_{xyz}$ and optical properties (extinction coefficient $K_{xyz}$, phase function parameters).
    *   LPM structure with spatial resolution $(r,s)$ and $D$ propagation directions.
    *   Current unscattered radiance $U_{d, p_{prev}, xyz}$ and average radiance $I_{d, xyz}$ buffers.
    *   Number of iterations $N_{iterations}$.
*   **Outputs:**
    *   Updated unscattered radiance $U_{d, p_{cur}, xyz}$ and average radiance $I_{d, xyz}$ for the next iteration.

### Data Structures
*   **Temporary Buffer ($T^i_{xyz}$):** A 3D buffer allocated for each propagation direction $i$. It stores the radiance contribution of rays traveling in direction $i$ as they traverse the volume. The size matches the volume grid $V_{xyz}$.
*   **LPM Elements:** Defined by coordinates $(r,s)$ and direction index $i$.
*   **GPU Blocks:**
    *   **Propagation Kernel:** One block per propagation direction $i$. Each block handles a subset of LPM elements $(r,s)$.
    *   **Collecting Kernel:** One block representing the entire volume. Each thread corresponds to one voxel $(x,y,z)$.

### Algorithm Steps

1.  **Initialization:**
    *   Set iteration index $p_{prev} = 0$.
    *   Initialize $U_{d, p_{prev}, xyz}$ and $I_{d, xyz}$ with self-emitted radiance (if any).

2.  **Iteration Loop ($p$ from $0$ to $N_{iterations}-1$):**
    *   Calculate current iteration index: $p_{cur} = (p_{prev} + 1) \pmod 2$.
    *   Initialize $U_{d, p_{cur}, xyz}$ to $0$.

    **A. Propagation Step:**
    *   For each LPM (or sub-LPM if resolution mismatch exists):
        *   **Parallel Execution:** Launch blocks for each propagation direction $i$.
            *   Within each block $i$, launch threads for each LPM element $(r,s)$ assigned to this block.
            *   **Thread Action:**
                1.  Create a ray with origin $(r,s)$ and direction $i$.
                2.  Initialize incoming radiance $L^{i,0}_{r,s}$ from $U_{d, p_{prev}, xyz}$ at the ray origin.
                3.  Trace the ray through the volume. For each intersected voxel $t$:
                    *   Compute the radiance contribution $L^{i,t}_{r,s}$ using the unscattered radiance from the previous iteration (Eq. 3 in [S1], typically involving transmittance and source terms).
                    *   Compute the temporary buffer value $T^i_{xyz}$ for the current voxel $(x,y,z)$ using Eq. 6:
                        $$T^i_{xyz} = L^{i,t}_{r,s} \left(1 - e^{-\Delta l_q K_{xyz} / \omega_z^n}\right)$$
                        Where:
                        *   $L^{i,t}_{r,s}$ is the radiance arriving at voxel $t$ from direction $i$.
                        *   $\Delta l_q$ is the step length.
                        *   $K_{xyz}$ is the extinction coefficient.
                        *   $\omega_z^n$ is the phase function parameter (often related to single scattering albedo or phase function evaluation).
                    *   Store $T^i_{xyz}$ into the temporary buffer for direction $i$.
                4.  **Intra-block Synchronization:** Use `__syncthreads()` whenever threads leave their current voxel to ensure consistent state within the block [S1].
        *   **Block Synchronization:** After all blocks for the current LPM have finished, synchronize all blocks (`Synchronize_blocks()`) before proceeding to the collecting step [S1].

    **B. Collecting Step:**
    *   Launch a single kernel block covering the entire volume.
    *   **Parallel Execution:** Each thread is assigned to one voxel $(x,y,z)$.
    *   **Thread Action:**
        1.  Initialize local accumulators for $U_{d, p_{cur}, xyz}$ and $I_{d, xyz}$ to $0$.
        2.  For each propagation direction $i$:
            *   Read the value $T^i_{xyz}$ from the temporary buffer for direction $i$.
            *   Update $U_{d, p_{cur}, xyz}$ and $I_{d, xyz}$ using the contributions from $T^i_{xyz}$ (Eq. 4 in [S1]). This typically involves adding the scattered light contribution to the average radiance and updating the unscattered radiance for the next bounce.
    *   Write the updated $U_{d, p_{cur}, xyz}$ and $I_{d, xyz}$ to global memory.

3.  **Update Index:**
    *   Set $p_{prev} = p_{cur}$.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Global variables
# volume: 3D grid of voxels with properties K_xyz, etc.
# lpm_elements: List of (r, s, direction_index)
# U_prev: 3D grid, unscattered radiance from previous iteration
# U_curr: 3D grid, unscattered radiance for current iteration
# I_curr: 3D grid, average radiance for current iteration
# T_buffers: Array of 3D grids, one per direction, temporary storage

def parallel_scalable_lpm(volume, lpm_elements, U_prev, I_curr, N_iterations):
    # Initialize U_curr and I_curr with self-emitted radiance if applicable
    # Note: I_curr is usually accumulated over bounces, U_prev is reset per bounce
    # For simplicity, assume U_prev is passed in, I_curr accumulates.
    
    p_prev = 0
    p_cur = 1
    
    for p in range(N_iterations):
        # Swap buffers
        U_curr = allocate_zeroed_grid(volume.size)
        # I_curr is accumulated, so we don't zero it, but we might need a temp buffer for accumulation
        I_accum = allocate_zeroed_grid(volume.size) 
        
        # --- Propagation Step ---
        # Group rays by direction
        directions = unique(lpm_elements.direction_index)
        
        for dir_idx in directions:
            # Get elements for this direction
            elements = [e for e in lpm_elements if e.direction_index == dir_idx]
            
            # Divide elements into blocks if necessary (see Block Synchronization)
            blocks = split_into_blocks(elements)
            
            # Launch Propagation Kernel for each block
            for block in blocks:
                launch_propagation_kernel(block, volume, U_prev, dir_idx, T_buffers[dir_idx])
            
            # Synchronize all propagation blocks
            synchronize_all_blocks()
            
        # --- Collecting Step ---
        # Launch Collecting Kernel
        launch_collecting_kernel(volume, T_buffers, U_curr, I_accum)
        
        # Update I_curr by adding I_accum
        I_curr = I_curr + I_accum
        
        # Prepare for next iteration
        U_prev = U_curr
        p_prev = p_cur
        p_cur = 1 - p_cur

def launch_propagation_kernel(block, volume, U_prev, dir_idx, T_buffer):
    # One thread per element (r,s) in the block
    for element in block.elements:
        r, s = element.r, element.s
        ray_origin = get_ray_origin(r, s, dir_idx)
        ray_direction = get_direction_vector(dir_idx)
        
        # Trace ray
        current_radiance = U_prev[ray_origin] # Simplified access
        
        for voxel in volume.intersected_by_ray(ray_origin, ray_direction):
            # Compute radiance contribution
            # Eq 3 logic: compute L^i,t_r,s from U_prev
            # This usually involves transmittance from previous voxel and source term
            transmittance = compute_transmittance(previous_voxel, voxel)
            source_term = compute_source_term(U_prev[voxel], volume.K[voxel])
            L_i_t = transmittance * current_radiance + source_term
            
            # Eq 6: Store contribution to temporary buffer
            delta_l = volume.step_length(voxel)
            K = volume.K[voxel]
            omega = volume.phase_param[voxel] # omega_z^n
            
            contribution = L_i_t * (1 - exp(-delta_l * K / omega))
            T_buffer[voxel] = contribution
            
            current_radiance = L_i_t # Update for next voxel
            
        # Intra-block sync if leaving voxel (handled by hardware/sync instruction in CUDA)
        # __syncthreads()

def launch_collecting_kernel(volume, T_buffers, U_curr, I_accum):
    # One thread per voxel
    for voxel in volume.all_voxels():
        u_val = 0.0
        i_val = 0.0
        
        for dir_idx in range(num_directions):
            t_val = T_buffers[dir_idx][voxel]
            
            # Eq 4: Update U and I
            # This step aggregates contributions from all directions
            # The exact formula depends on the specific LPM variant's Eq 4
            # Typically: I_new = I_old + sum(T^i)
            # And U is updated for the next bounce based on scattering
            i_val += t_val
            # U update logic is specific to the scattering model
            # Assuming U_curr is reset to 0, we might need to compute it from I or vice versa
            # Based on Alg 2, U is initialized to 0, then updated.
            # Let's assume U_curr accumulates scattered light for next iteration
            u_val += t_val 
            
        U_curr[voxel] = u_val
        I_accum[voxel] = i_val
```

### Parameters and Typical Values
*   **LPM Resolution:** Often matches the volume face resolution to simplify mapping [S1]. If LPM resolution is $N$ times higher than voxel grid, divide into $N$ sub-LPMs [S1].
*   **Block Size:** Limited by register count. Reduced by lowering resolution of one LPM coordinate ($r$ or $s$), increasing the number of blocks per direction [S1].
*   **Streaming:** For large volumes, sub-volumes (e.g., $16 \times 16 \times 16$) are processed sequentially to fit in GPU memory [S1].

### Edge Cases and Pitfalls
*   **Synchronization Overhead:** Naive parallelization with atomic operations is inefficient due to branching and global memory transactions [S1]. The proposed method avoids this via data locality [S1].
*   **Boundary Ray Collision:** When multiple blocks propagate light in the same direction, voxels at the frontier between contiguous subsets of LPM elements ($R_i$ and $R_{i+1}$) may be traversed by two rays simultaneously. This is mitigated by launching blocks $B_{2i}$ and $B_{2i+1}$ in parallel, but not contiguous blocks $B_i$ and $B_{i+1}$ [S1].
*   **Thread Limits:** The number of threads per block is limited by registers. If the block size exceeds this limit, the resolution of one LPM coordinate must be reduced [S1].
*   **Resolution Mismatch:** If LPM resolution is significantly higher than voxel grid, synchronization problems arise. Solution: Divide LPM into $N$ sub-LPMs and propagate sequentially [S1].

## Subtopics

### Block Synchronization and Launch Strategy
This variant focuses on the specific launch configuration to prevent race conditions at block boundaries. To guarantee only one thread per voxel, blocks $B_{2i}$ are launched in parallel, followed by $B_{2i+1}$, avoiding the simultaneous launch of contiguous blocks $B_i$ and $B_{i+1}$ which could cause frontier voxel collisions [S1]. Intra-block synchronization using `__syncthreads()` is required whenever rays leave their current voxel [S1]. If the LPM resolution is $N$ times higher than the voxel grid, the LPM is divided into $N$ smaller sub-LPMs propagated sequentially to avoid synchronization issues [S1].

## Operating characteristics
*   **Time Complexity:** The method scales with the number of rays (LPM elements) and volume size. The speedup over the original CPU algorithm increases with volume size because more blocks are created, keeping GPU resources busy [S1].
*   **Memory Complexity:** Memory usage is dominated by the temporary buffers $T^i_{xyz}$ for each direction. For large volumes, streaming is used to process sub-volumes, keeping memory usage far below available GPU memory (e.g., handling volumes larger than $256^3$) [S1].
*   **Performance:** On a GTX 580, the parallel approach shows significant speedup over atomic-operation-based versions and the original CPU algorithm. For a $64^3$ volume with 25 propagation directions and 3 bounces, it takes 417 milliseconds [S1].
*   **Strengths:** High efficiency due to data locality (cache accesses) instead of global memory transactions. Scalable to large volumes via streaming.
*   **Limitations:** Requires careful management of block launches and synchronization. Memory usage increases with volume size, though at a slower rate with streaming [S1].

## Background
Parallel and Scalable LPM was introduced by [S1] to address the inefficiencies of naive GPU implementations of Fattal's LPM algorithm. The original algorithm suffered from synchronization problems when multiple rays wrote to the same voxel. The authors proposed the two-step propagation/collecting approach to leverage GPU parallelism while maintaining correctness and performance. The method is part of the broader family of Light Propagation Maps, which are used for efficient light transport in participating media.

## Sources
[S1] Light Propagation Maps on Parallel Graphics Architectures, https://doi.org/10.2312/egpgv/egpgv12/081-088, fetched 2026-10-26
