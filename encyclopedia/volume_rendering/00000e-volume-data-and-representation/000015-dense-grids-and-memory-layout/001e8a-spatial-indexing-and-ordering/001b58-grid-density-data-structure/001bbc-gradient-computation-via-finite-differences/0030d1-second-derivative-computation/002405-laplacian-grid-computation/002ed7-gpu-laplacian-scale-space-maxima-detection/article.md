# GPU Laplacian Scale-Space Maxima Detection
Also called: GPU Scale-Space Computation, GPU Scale-Space Computation and Maxima Detection, Laplacian pyramid computation, Per-iteration scale detection, Scale abstraction generation

## Overview
This method detects local maxima of the Laplacian $\nabla^2 f$ across a scale space to identify blob-like structures in volumetric data, encoding the results as a scale abstraction [S1][S2, unofficial]. It differs from sibling methods like [Scale Field Computation](6538-scale-field-computation.md) by merging the scale-space diffusion and maxima detection into a single per-iteration pass, avoiding the storage of the full scale-space pyramid in GPU memory [S1]. This approach solves the memory bottleneck associated with storing full-resolution scale fields for large volumes [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume texture representing the current iteration's scalar field.
*   **Output:** A set of 3D coordinates (and associated scale values) denoting scale maxima, stored as the "scale abstraction" on the CPU [S1][S2, unofficial].

**Data Structures:**
The algorithm uses a 3D texture for the scalar field and a 4-channel pixel buffer (RGBA) to encode state for maxima detection. The pixel encoding layout is fixed as follows [S1][S2, unofficial]:
*   **Red (R):** $L(x; t+1)$ — The scalar value at the next iteration.
*   **Green (G):** $\nabla^2 L(x; t+1)$ — The Laplacian at the next iteration.
*   **Blue (B):** $\nabla^2 L(x; t)$ — The Laplacian at the current iteration.
*   **Alpha (A):** $\nabla^2 L(x; t-1)$ — The Laplacian at the previous iteration.

Here, $x$ is the spatial coordinate, $t$ is the current iteration index (scale level), and $L(x;t)$ is the diffused scalar value at iteration $t$.

**Algorithm Steps:**
1.  **Initialization:** Load the input volume into a 3D texture. Initialize the iteration counter $t = 0$.
2.  **Diffusion and Derivative Computation:** For each active voxel, solve the diffusion equation (Eq. 4 in [S1]) to compute the new scalar value $L(x; t+1)$ and its second spatial derivatives (Laplacian $\nabla^2 L(x; t+1)$) [S1][S2, unofficial]. This is implemented in a pixel shader [S1].
3.  **State Encoding:** Write the computed values into the 4-channel pixel buffer according to the layout defined above. The previous iteration's Laplacian ($\nabla^2 L(x; t-1)$) is already present in the Alpha channel from the previous step, and the current Laplacian ($\nabla^2 L(x; t)$) is in the Blue channel [S1][S2, unofficial].
4.  **Maxima Detection:** Compare the current Laplacian value (Blue channel) against the Laplacian values of its spatial neighbors at iterations $t-1$ (Alpha), $t$ (Blue), and $t+1$ (Green) [S1][S2, unofficial].
    *   If $\nabla^2 L(x; t)$ is the maximum among these values, the pixel is identified as a scale maximum.
    *   Write the coordinate and scale information to an output buffer [S1][S2, unofficial].
5.  **Ping-Pong Swap:** Use a ping-pong approach with two 3D textures to avoid simultaneous read/write conflicts on the same texture slice [S1][S2, unofficial]. The output of the current iteration becomes the input for the next.
6.  **Iteration:** Increment $t$ and repeat steps 2–5 until the desired scale range is covered [S1][S2, unofficial].
7.  **Readback:** Read the pixels denoting scale maxima back to the CPU and store them as the scale abstraction [S1][S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Assume input_volume is a 3D texture
# output_maxima is a list of (x, y, z, t) tuples
# scale_field is a 3D texture used for ping-pong buffering
# prev_laplacian_texture is a 3D texture storing Laplacian values from t-1

t = 0
# Initialize prev_laplacian_texture with zeros or boundary conditions
# Initialize scale_field with input_volume

while t < max_iterations:
    # Create temporary textures for current and next iteration
    curr_laplacian_texture = new_3d_texture()
    next_scalar_texture = new_3d_texture()
    
    # Process each voxel in parallel (simulated here as a loop)
    for each voxel x in volume:
        # 1. Solve diffusion equation to get L(x; t+1)
        # Eq. 4 from [S1] implies a Gaussian blur or heat equation step
        L_next = solve_diffusion(scale_field[x], prev_laplacian_texture[x])
        
        # 2. Compute Laplacian at t+1
        # Using 6-point stencil from parent [Laplacian Grid Computation](3160-laplacian-grid-computation.md)
        Lap_next = compute_laplacian(next_scalar_texture, x)
        
        # 3. Retrieve Laplacian at t (current) and t-1 (previous)
        # Note: In the first iteration, Lap_current might be the Laplacian of the input
        Lap_current = compute_laplacian(scale_field, x)
        Lap_prev = prev_laplacian_texture[x]
        
        # 4. Encode into RGBA buffer for next step's comparison
        # R: L(x; t+1), G: Lap_next, B: Lap_current, A: Lap_prev
        pixel_buffer[x] = (L_next, Lap_next, Lap_current, Lap_prev)
        
        # 5. Maxima Detection
        # Check if Lap_current is max among neighbors in t-1, t, t+1
        # Neighbors in space are checked against their respective temporal Laplacians
        is_max = True
        for neighbor in spatial_neighbors(x):
            # Get Laplacian of neighbor at t-1 (Alpha), t (Blue), t+1 (Green)
            # Note: The source [S1][S2, unofficial] implies comparing the center pixel's 
            # current Laplacian against the neighbors' Laplacians across time.
            # Specifically: "maximum of the Laplacian of the neighbors in iterations t-1, t and t+1"
            # This phrasing is slightly ambiguous. Standard scale-space maxima detection 
            # compares L(x,t) against L(x',t) for spatial neighbors x' and L(x,t') for temporal neighbors t'.
            # The source says: "value of the Laplacian at a given iteration (Blue) is the maximum 
            # of the Laplacian of the neighbors in iterations t-1, t and t+1 (GBA channels)".
            # This implies a 3D spatial neighborhood check where each neighbor's temporal history is available.
            
            # Retrieve neighbor's Laplacians from the encoded buffer of the neighbor
            n_Lap_t_minus_1 = pixel_buffer[neighbor].A
            n_Lap_t = pixel_buffer[neighbor].B
            n_Lap_t_plus_1 = pixel_buffer[neighbor].G
            
            # The center pixel's current Laplacian is pixel_buffer[x].B
            # The condition is that pixel_buffer[x].B is the maximum among 
            # the set of Laplacians: {n_Lap_t_minus_1, n_Lap_t, n_Lap_t_plus_1} for all spatial neighbors?
            # Or does it mean L(x,t) > L(x_neighbor, t) AND L(x,t) > L(x, t+1) AND L(x,t) > L(x, t-1)?
            # Source [S1]: "maximum of the Laplacian of the neighbors in iterations t-1, t and t+1"
            # This suggests comparing the center's current Laplacian against the neighbors' Laplacians at all three time steps.
            
            if pixel_buffer[x].B < n_Lap_t_minus_1 or \
               pixel_buffer[x].B < n_Lap_t or \
               pixel_buffer[x].B < n_Lap_t_plus_1:
                is_max = False
                break
        
        if is_max:
            output_maxima.append((x, t))
            
    # Update textures for next iteration
    prev_laplacian_texture = curr_laplacian_texture # Or store Lap_current
    scale_field = next_scalar_texture
    t += 1

# Readback output_maxima to CPU
```
*[G] The pseudocode clarifies the "neighbors in iterations" logic. Standard scale-space maxima detection usually compares $L(x,t)$ against $L(x',t)$ for spatial neighbors $x'$ and $L(x,t')$ for temporal neighbors $t'$. The source text explicitly mentions comparing the Blue channel (current Laplacian) against GBA channels (next, current, previous Laplacians) of the *neighbors*. This implies a 3D spatial neighborhood where each neighbor's temporal history is used. If the source meant comparing the center pixel's temporal history against its spatial neighbors' current values, the encoding would be different. Given the explicit channel mapping, the implementation checks if the center's current Laplacian is greater than the neighbors' Laplacians at $t-1, t, t+1$.]*

**Parameters:**
*   $t$: Iteration variable representing the current scale level [S2, unofficial].
*   Resolution: The scale field may be computed at a lower resolution (e.g., subsampled by a factor of 4) to fit in GPU memory [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Memory Limitations:** A full-precision $512^3$ scale field cannot fit in current GPU texture memory [S2, unofficial]. Solution: Compute at lower resolution or analyze sub-regions [S2, unofficial].
*   **Resolution Limits:** Subsampling by a factor of 4 restricts detectable sizes; features smaller than the subsampling factor (e.g., 1-4 voxels) cannot be discerned [S2, unofficial].
*   **Simultaneous Read/Write:** Requires a ping-pong approach to avoid conflicts [S1][S2, unofficial].

## Subtopics

### Feature Detectors in Scale Space
A method to detect blobs by analyzing the behavior of derivatives of the scale space along the scale dimension, specifically detecting Laplacian maxima in both scale and space [S1]. The scale $t$ at which the blob is detected serves as the representative scale of the blob, allowing a Gaussian blob of width $t$ centered at the detected point to describe the structure's size and location [S1].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of voxels and iterations, as each iteration processes the volume in a single pass [S1].
*   **Memory Complexity:** $O(N)$ where $N$ is the number of voxels, as only one temporary volume is stored at a time, avoiding the $O(N \times S)$ cost of storing the full scale space ($S$ scales) [S1].
*   **Performance:** Implemented on an nVidia GeForce 8800 GTX with an Intel Core 2 Duo 2.4 GHz CPU [S1].
*   **Strengths:** Real-time capability; avoids prohibitive memory usage for large volumes [S1][S2, unofficial].
*   **Limitations:** Detectable sizes are restricted by the resolution of the scale field if subsampled; sub-voxel detection requires analyzing sub-regions [S2, unofficial].

## Background
The method is based on the work of Correa and Silva (2008) in "Size-based Transfer Functions: A New Volume Exploration Technique" [S1][S2, unofficial]. It builds on the principle that feature detectors can be obtained by looking at the derivatives of the scale space along the scale dimension [S1]. Unlike pyramids which subsample space and scale, this method maintains spatial resolution while varying scale through diffusion [S1].

## Sources
[S1] Size-based Transfer Functions: A New Volume Exploration Technique, http://carloscorrea.com/docs/vis08.pdf, fetched 2026-10-29
[S2] Size-based Transfer Functions: A New Volume Exploration Technique, https://www.researchgate.net/publication/23456199_Size-based_Transfer_Functions_A_New_Volume_Exploration_Technique, fetched 2026-10-30 (unofficial)
