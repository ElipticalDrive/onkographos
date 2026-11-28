# Macrocell Grid Traversal with Majorant Densities

Also called: 3D-DDA with Woodcock tracking, Majorant-based free-flight tracking, Single-Level Macrocell + DDA Space Partitioning

## Overview
This method accelerates volumetric path tracing by combining a coarse spatial partition (macrocells) with Delta Tracking (Woodcock tracking) to skip empty or low-opacity regions more efficiently than fixed-step marching [S1]. It sets itself apart from its parent [Adaptive Ray Marching with Macrocell DDA](1500-adaptive-ray-marching-with-macrocell-dda.md) by explicitly using the alpha component of the transfer function as a majorant extinction coefficient to determine probabilistic free-flight distances, allowing larger steps in homogeneous cells while maintaining unbiased Monte Carlo estimation [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Ray origin $\mathbf{o}$ and direction $\mathbf{d}$ in world space; a transfer function $TF$ mapping density to RGBA; a macrocell grid storing min/max density ranges for each cell [S1].
*   **Output:** Accumulated radiance contribution from the volume along the ray [S1].

**Data Structures:**
The spatial partition is defined by `DeviceSpacePartiton_SingleMC` [S2, unofficial]:
*   `value_ranges`: Pointer to `range1f*`, storing min/max density values for each macrocell [S2, unofficial].
*   `majorants`: Pointer to `float*`, storing the precomputed majorant extinction coefficient for each macrocell [S2, unofficial].
*   `dims`: `vec3i`, dimensions of the macrocell grid in cells [S2, unofficial].
*   `spac`: `vec3f`, spatial scaling factor (world units per macrocell) [S2, unofficial].
*   `spac_rcp`: `vec3f`, reciprocal of `spac`, used to convert world coordinates to macrocell grid coordinates [S2, unofficial].
*   Macrocell size is fixed at $16$ ($1 \ll 4$), where $4$ is `MACROCELL_SIZE_MIP` [S2, unofficial].

**Algorithm Steps:**

1.  **Initialization:** Transform the ray origin and direction into macrocell space using the reciprocal scaling factor: $\mathbf{m\_org} = \mathbf{o} \cdot \mathbf{spac\_rcp}$ and $\mathbf{m\_dir} = \mathbf{d} \cdot \mathbf{spac\_rcp}$ [S2, unofficial]. Initialize the 3D-DDA iterator with $\mathbf{m\_org}$, $\mathbf{m\_dir}$, and the ray's near/far bounds $[t_{near}, t_{far}]$ in macrocell space [S2, unofficial].
2.  **Majorant Computation:** For each macrocell, retrieve the stored min/max density range $[d_{min}, d_{max}]$ [S1]. Look up these values in the transfer function $TF$ to obtain the alpha components $\alpha_{min} = TF(d_{min}).a$ and $\alpha_{max} = TF(d_{max}).a$ [S1]. The majorant extinction coefficient $\sigma_t$ for the cell is derived from these values [S1]. Majorants must be recomputed whenever the transfer function changes [S1]. In the reference implementation, fixed constants are used for demonstration: $\sigma_t = 1.0$ and density scale $= 1.0$ [S2, unofficial].
3.  **Traversal Loop:** While the ray has not exited the grid ($t < t_{far}$):
    a. **Cell Access:** Determine the current macrocell index $idx$ from the DDA iterator's current cell coordinates $(x, y, z)$: $idx = x + y \cdot dims.x + z \cdot dims.x \cdot dims.y$ [S2, unofficial].
    b. **Empty Cell Skip:** If the cell is empty (e.g., $d_{max} = 0$ or $\sigma_t = 0$), advance the DDA iterator to the next intersected cell and continue to step 3a [S1].
    c. **Woodcock Tracking:** If the cell is non-empty, perform Delta Tracking within the current cell [S1]:
       i. Compute the distance to the next cell boundary $t_{cell\_end}$ using the DDA state [G].
       ii. Sample a free-flight distance $t_{ff}$ from the exponential distribution with rate $\sigma_t$: $t_{ff} = -\frac{\ln(1 - \xi)}{\sigma_t}$, where $\xi$ is a uniform random number in $[0, 1)$ [G].
       iii. Determine the step size $t_{step} = \min(t_{ff}, t_{cell\_end} - t_{current})$ [G].
       iv. Advance the ray position: $\mathbf{p} = \mathbf{o} + (t_{current} + t_{step}) \mathbf{d}$ [G].
       v. **Interaction Check:** If $t_{step} \approx t_{ff}$, an interaction occurred [G]. Query the actual density at $\mathbf{p}$ and compute the actual extinction coefficient $\sigma_a$ from the transfer function [G]. If $\sigma_a / \sigma_t$ passes a random test, scatter or absorb; otherwise, treat as a virtual interaction and continue [G].
       vi. If $t_{step} \approx t_{cell\_end}$, the ray reached the cell boundary [G]. Update $t_{current}$ and advance the DDA iterator to the next cell [S1].
4.  **Termination:** If the ray exits the grid ($t \ge t_{far}$) or is canceled by Russian Roulette during scattering, return the accumulated radiance [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def macrocell_traversal_with_majorants(ray_o, ray_d, tnear, tfar, macrocell_grid, transfer_function):
    """
    ray_o: list of 3 floats, ray origin in world space
    ray_d: list of 3 floats, ray direction in world space
    tnear: float, near bound in macrocell space
    tfar: float, far bound in macrocell space
    macrocell_grid: dict with keys 'dims', 'spac_rcp', 'majorants'
    transfer_function: callable mapping density to RGBA tuple
    
    Returns: accumulated radiance (list of 3 floats)
    """
    import math
    import random

    spac_rcp = macrocell_grid['spac_rcp']
    dims = macrocell_grid['dims']
    majorants = macrocell_grid['majorants']
    
    # 1. Transform to macrocell space
    m_org = [ray_o[i] * spac_rcp[i] for i in range(3)]
    m_dir = [ray_d[i] * spac_rcp[i] for i in range(3)]
    
    # Initialize 3D-DDA state
    dda_x, dda_y, dda_z = int(m_org[0]), int(m_org[1]), int(m_org[2])
    dda_dx = 1 if m_dir[0] > 0 else -1
    dda_dy = 1 if m_dir[1] > 0 else -1
    dda_dz = 1 if m_dir[2] > 0 else -1
    
    # Distance to next boundary in macrocell space
    t_max_x = (dda_x + (1.0 if dda_dx > 0 else 0.0) - m_org[0]) / m_dir[0] if m_dir[0] != 0 else float('inf')
    t_max_y = (dda_y + (1.0 if dda_dy > 0 else 0.0) - m_org[1]) / m_dir[1] if m_dir[1] != 0 else float('inf')
    t_max_z = (dda_z + (1.0 if dda_dz > 0 else 0.0) - m_org[2]) / m_dir[2] if m_dir[2] != 0 else float('inf')
    
    t_delta_x = dda_dx / m_dir[0] if m_dir[0] != 0 else float('inf')
    t_delta_y = dda_dy / m_dir[1] if m_dir[1] != 0 else float('inf')
    t_delta_z = dda_dz / m_dir[2] if m_dir[2] != 0 else float('inf')
    
    accumulated_radiance = [0.0, 0.0, 0.0]
    t_current = tnear
    
    while t_current < tfar:
        # Check bounds to prevent index out of range
        if not (0 <= dda_x < dims[0] and 0 <= dda_y < dims[1] and 0 <= dda_z < dims[2]):
            break
            
        idx = dda_x + dda_y * dims[0] + dda_z * dims[0] * dims[1]
        # Reference implementation uses fixed constant for demonstration [S2, unofficial]
        sigma_t = majorants[idx] if majorants else 1.0 
        
        # Skip empty cells
        if sigma_t <= 0.0:
            # Advance DDA to next cell
            if t_max_x < t_max_y:
                if t_max_x < t_max_z:
                    dda_x += dda_dx; t_max_x += t_delta_x
                else:
                    dda_z += dda_dz; t_max_z += t_delta_z
            else:
                if t_max_y < t_max_z:
                    dda_y += dda_dy; t_max_y += t_delta_y
                else:
                    dda_z += dda_dz; t_max_z += t_delta_z
            t_current = min(t_max_x, t_max_y, t_max_z)
            continue
            
        # Woodcock Tracking within the cell
        t_cell_end = min(t_max_x, t_max_y, t_max_z)
        
        # Guard against floating point drift on boundaries
        if t_cell_end <= t_current:
            if t_max_x < t_max_y:
                if t_max_x < t_max_z:
                    dda_x += dda_dx; t_max_x += t_delta_x
                else:
                    dda_z += dda_dz; t_max_z += t_delta_z
            else:
                if t_max_y < t_max_z:
                    dda_y += dda_dy; t_max_y += t_delta_y
                else:
                    dda_z += dda_dz; t_max_z += t_delta_z
            t_current = min(t_max_x, t_max_y, t_max_z)
            continue
            
        # Sample free-flight distance [G]
        xi = random.random()
        t_ff = -math.log(1.0 - xi) / sigma_t
        
        # Determine step size [G]
        t_step = min(t_ff, t_cell_end - t_current)
        
        # Advance ray [G]
        t_current += t_step
        p_world = [ray_o[i] + t_current * ray_d[i] for i in range(3)]
        
        # Interaction Check: If t_step approx t_ff, an interaction occurred [G]
        if t_step >= t_ff - 1e-6:
            # Density query not given in the sources [G]
            density = 0.0 
            alpha_actual = transfer_function(density)[3]
            sigma_a = alpha_actual
            
            # Russian Roulette for scattering vs absorption [G]
            if sigma_t > 0 and random.random() < sigma_a / sigma_t:
                # Scattering event: contribute radiance and potentially spawn a new ray [G]
                accumulated_radiance = [0.1, 0.2, 0.3] # Placeholder for actual contribution [G]
                break # Path terminated or continued
            else:
                # Virtual interaction, continue tracking [G]
                pass
        else:
            # Reached cell boundary, advance DDA [S1]
            if t_max_x < t_max_y:
                if t_max_x < t_max_z:
                    dda_x += dda_dx; t_max_x += t_delta_x
                else:
                    dda_z += dda_dz; t_max_z += t_delta_z
            else:
                if t_max_y < t_max_z:
                    dda_y += dda_dy; t_max_y += t_delta_y
                else:
                    dda_z += dda_dz; t_max_z += t_delta_z
                    
    return accumulated_radiance
```

**Parameters:**
*   `MACROCELL_SIZE_MIP`: Log2 of macrocell dimension, typically 4 (size 16) [S2, unofficial].
*   `density_scale`: Scaling factor for density values, typically 1.0 [S2, unofficial].
*   `sigma_t`: Majorant extinction coefficient, typically 1.0 in basic implementations [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Transfer Function Changes:** Majorants must be recomputed whenever the transfer function changes, as they depend on the min/max density ranges [S1].
*   **Homogeneity Assumption:** Woodcock tracking is most efficient in homogeneous cells where the majorant closely approximates the actual density [S1].
*   **Hierarchical Diminishing Returns:** Experiments with hierarchical space skipping showed diminishing returns compared to uniform grids when using RGBA transfer functions, likely due to the overhead of managing multiple levels of majorants [S1].

## Subtopics

### [Delta Tracking with DDA Iteration (Macrocell-based)](1534-delta-tracking-with-dda-iteration-macrocell-based.md)
A ray-marching technique for volume rendering that combines a Discrete Differential Analyzer (DDA) for traversing a coarse spatial partition (macrocells) with Delta Tracking (Russian Roulette) for sampling absorption and scattering events within those cells. It sets itself apart from standard DDA by replacing uniform step sizes with probabilistic free-flight distances derived from the majorant, allowing it to skip non-interacting regions within a macrocell more efficiently than fixed-step marching, while avoiding the overhead of full per-voxel traversal [G]. It is chosen when the volume is sparse or heterogeneous, and the transfer function allows for effective majorant estimation [G]. See [Delta Tracking with DDA Iteration (Macrocell-based)](1534-delta-tracking-with-dda-iteration-macrocell-based.md).

### ANARI Renderer Integration
A renderer using the ANARI API that integrates VDB optimization. It sets itself apart by hiding compression details behind an ANSI-C API, converting ANARI structured-regular fields to NanoVDB grids internally, making the optimization transparent to the user. It is chosen when interoperability with standard visualization tools like ParaView is required. See [ANARI Renderer Integration](1535-anari-renderer-integration.md).

## Operating characteristics
*   **Time Complexity:** $O(N_{cells} + N_{interactions})$, where $N_{cells}$ is the number of macrocells traversed and $N_{interactions}$ is the number of scattering events [G]. The DDA traversal is linear in the number of cells, and Woodcock tracking is efficient in sparse regions [G].
*   **Memory Complexity:** $O(N_{macrocells})$ for storing the majorant grid and min/max ranges [G]. This is significantly smaller than a full voxel grid for sparse volumes [G].
*   **Measured Performance:** In the reference implementation (Barney renderer), this method reduces sample count in homogeneous regions by allowing larger steps via Woodcock tracking [S1]. Performance gains are most significant in sparse volumes where empty cells can be skipped entirely [S1].
*   **Strengths:** Efficiently skips empty regions; adapts step size to local homogeneity; unbiased Monte Carlo estimation; simple uniform grid structure avoids hierarchical overhead [S1].
*   **Limitations:** Requires recomputation of majorants if transfer function changes [S1].
*   **When to Use:** Sparse or heterogeneous volumes with well-defined transfer functions; when unbiased path tracing is required [G].
*   **When Not to Use:** Dense volumes where the overhead of macrocell traversal outweighs benefits; when hierarchical structures are preferred (though diminishing returns were observed) [S1].

## Background
This method originates from the integration of VDB optimization into the Barney renderer, a multi-GPU wavefront path tracer [S1]. The approach builds on earlier work in 3D-DDA for volume rendering [SKTM11] and combines it with Woodcock tracking for free-flight distance sampling [S1]. The use of a uniform grid instead of hierarchical structures was chosen due to diminishing returns observed with hierarchical space skipping in the presence of RGBA transfer functions [S1].

## Sources
[S1] GPU Volume Rendering with Hierarchical Compression Using VDB, https://arxiv.org/pdf/2504.04564, fetched 2026-10-26
[S2] GitHub - VIDILabs/open-volume-renderer: Open source modular multi ..., https://github.com/VIDILabs/open-volume-renderer, fetched 2026-10-29 (unofficial)
