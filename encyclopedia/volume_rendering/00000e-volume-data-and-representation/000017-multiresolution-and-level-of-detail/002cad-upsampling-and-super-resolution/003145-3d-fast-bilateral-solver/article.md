# 3D Fast Bilateral Solver

Also called: 3D FBS, 3D Bilateral Solver

## Overview
The 3D Fast Bilateral Solver (3D FBS) is a post-processing refinement technique used to upsample low-resolution similarity volumes to high resolution while adapting them to the edges of an underlying raw intensity volume [S1]. It solves the problem of visual artifacts and loss of fine detail that occur when rendering segmentation or similarity maps generated at coarse resolutions [S1]. Unlike simple interpolation, it uses edge-aware smoothing guided by the raw volume's intensity gradients to restore voxel-precise borders [S1]. It differs from 2D propagation methods by operating directly in 3D, avoiding complex topology solving and multi-second inference times associated with 3D neural network inference [S1].

## How it works
The method takes a low-resolution similarity map $S_L$ and a raw intensity volume $V$ as inputs, and produces a high-resolution, edge-aligned similarity map $S_H$ [S1]. The core mechanism extends the 2D Fast Bilateral Solver to 3D by incorporating a z-component into the bilateral grid vertices, allowing the solver to consider depth information during the smoothing process [S1].

### Data Structures
*   **Low-Resolution Similarity Map ($S_L$):** A 3D volume of similarity values, typically at a resolution significantly lower than the target (e.g., $64^3$ to $96^3$) [S1].
*   **Raw Intensity Volume ($V$):** The input volume containing the actual intensity data (e.g., CT scan), serving as the reference for edge detection [S1].
*   **High-Resolution Similarity Map ($S_H$):** The output volume, typically at $256^3$ or $512^3$ resolution [S1].
*   **Bilateral Grid:** A 3D grid used to store the solution. The solver extends the 2D approach by adding a z-coordinate to each vertex in the bilateral grid, enabling 3D spatial smoothing [S1].

### Parameters
*   **Threshold ($\tau$):** A value used to crop low-similarity regions. The default empirical value is $\tau = 0.25$ [S1].
*   **Bilateral Solver $\sigma$ parameters:** Control the window size used for blurring. These parameters influence the occurrence of block artifacts if the initial detection in $S_L$ is insufficient [S1].
*   **Output Resolution:** The target resolution for $S_H$, typically $256^3$ or $512^3$, chosen based on the class and actual size of the structure to balance quality and speed [S1].

### Steps
1.  **Upsampling:** Upsample the low-resolution similarity map $S_L$ tri-linearly to match the resolution of the raw volume $V$ [S1].
2.  **Cropping:** Crop the regions where the similarity value $S > \tau$ to discard low-similarity areas. This reduces the computational load and focuses the solver on relevant structures [S1].
3.  **Solving:** Solve for the smoothed high-resolution similarity $S_H$ using the cropped region from $V$ as a reference for edge-awareness. The 3D extension adds a z-component to the bilateral grid vertices to handle 3D spatial relationships [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def three_d_fast_bilateral_solver(S_L, V, tau=0.25, target_res=(256, 256, 256)):
    """
    Refines a low-resolution similarity map using the 3D Fast Bilateral Solver.
    
    Args:
        S_L: Low-resolution similarity volume.
        V: Raw intensity volume (reference for edges).
        tau: Threshold for cropping low-similarity regions.
        target_res: Target resolution for the output volume.
        
    Returns:
        S_H: High-resolution, edge-aligned similarity volume.
    """
    # Step 1: Upsample S_L tri-linearly to match the resolution of V
    # Note: The source specifies matching V's resolution for upsampling,
    # but the final output S_H can be at a different resolution (target_res).
    # We assume V is at the resolution of interest for edge reference.
    S_upsampled = tri_linear_upsample(S_L, V.shape)
    
    # Step 2: Crop regions where similarity S > tau
    # Create a mask for valid regions
    mask = S_upsampled > tau
    
    # Crop the volume to the bounding box of the mask
    # This reduces the size of the bilateral grid, speeding up the solver
    cropped_S, cropped_V, crop_bounds = crop_volume(S_upsampled, V, mask, threshold=tau)
    
    # Step 3: Solve for smoothed S_H using the cropped region from V as reference
    # The 3D extension adds a z-component to the bilateral grid vertices
    # The solver minimizes an energy function that balances fidelity to S_upsampled
    # and smoothness guided by the gradients in V.
    S_H_cropped = solve_bilateral_solver_3d(cropped_S, cropped_V)
    
    # Place the result back into the full volume at the target resolution
    # If target_res differs from V's resolution, further upsampling or resizing may be needed
    S_H = resize_to_target(S_H_cropped, crop_bounds, target_res)
    
    return S_H

def tri_linear_upsample(volume, target_shape):
    """
    Performs tri-linear interpolation to upsample a 3D volume.
    """
    # Implementation of tri-linear interpolation
    # For each voxel in target_shape, calculate the weighted average of the 8 nearest neighbors in volume
    upscaled = np.zeros(target_shape)
    # ... interpolation logic ...
    return upscaled

def crop_volume(S, V, mask, threshold):
    """
    Crops volumes to the bounding box of the mask where S > threshold.
    """
    # Find indices where mask is True
    indices = np.where(mask)
    if not indices[0].size:
        return None, None, None
    
    # Calculate bounding box
    min_coords = np.min(indices, axis=1)
    max_coords = np.max(indices, axis=1)
    
    # Crop S and V
    cropped_S = S[min_coords[0]:max_coords[0]+1, min_coords[1]:max_coords[1]+1, min_coords[2]:max_coords[2]+1]
    cropped_V = V[min_coords[0]:max_coords[0]+1, min_coords[1]:max_coords[1]+1, min_coords[2]:max_coords[2]+1]
    
    return cropped_S, cropped_V, (min_coords, max_coords)

def solve_bilateral_solver_3d(S_cropped, V_cropped):
    """
    Solves the 3D Fast Bilateral Solver.
    This involves constructing a bilateral grid with z-components and solving a linear system.
    """
    # Construct bilateral grid with z-component
    # The grid vertices are placed in a higher-dimensional space including spatial (x,y,z) and intensity (V) dimensions
    # The exact construction of the bilateral grid and the linear system solution is based on the Fast Bilateral Solver [12]
    # extended to 3D.
    # ... solver implementation ...
    return S_smoothed
```

### Edge Cases and Pitfalls
*   **Insufficient Detection:** The bilateral solver cannot complete structures far beyond what is detected in $S_L$. If the structure is not sufficiently detected in the low-resolution similarity map, the refined similarity falls off smoothly towards the surface, resulting in block artifacts when rendered as an iso-surface [S1].
*   **Block Artifacts:** These arise from the $\sigma$ parameters of the bilateral solver, which control the window used for blurring. They occur specifically when the structure is not sufficiently detected in $S_L$ [S1].
*   **Connected Components Filter:** Using a connected components filter to discard falsely detected disconnected components fails when target structures are too close together [S1].
*   **Resolution Dependency:** Post-processing time varies with resolution, unlike the initial inference time [S1].

## Operating characteristics
*   **Time Complexity:** The post-processing time varies with resolution. On an Intel i7-8700K, it takes approximately 0.4 seconds for a $256^3$ volume and 5.3 seconds for a $512^3$ volume [S1].
*   **Memory Complexity:** The method requires VRAM proportional to the resolution of the feature extraction and the bilateral grid. 8GB of VRAM suffices for feature extraction at $80^3$ resolution, while higher resolutions demand prohibitive amounts of VRAM [S1].
*   **Strengths:** It effectively aligns low-resolution feature maps to the raw input, especially at borders, and restores fine details lost due to low resolution in $S_L$ [S1]. It is fast enough for interactive use (approx. 0.5 seconds) [S1].
*   **Limitations:** It requires sufficient detection of the full structure in $S_L$ to produce surfaces without artifacts. It cannot complete structures that are not detected in the initial map [S1].
*   **When to Use:** Use when you have a low-resolution similarity map and need to refine it to match the edges of a high-resolution intensity volume for better visual quality in rendering [S1].
*   **When Not to Use:** Do not use if the initial similarity map $S_L$ lacks sufficient detection of the structure, as this will lead to block artifacts [S1].

## Background
The 3D Fast Bilateral Solver is a 3D extension of the Fast Bilateral Solver [12], proposed in the paper "Leveraging Self-Supervised Vision Transformers for Segmentation-Based Transfer Function Design" [S1]. It was introduced to address the issue of low visual quality when rendering similarity maps directly, by increasing their resolution and adapting them to the underlying intensity volume [S1]. The method is part of a workflow that includes feature extraction using a frozen self-supervised Vision Transformer and interactive annotation [S1].

## Sources
[S1] Leveraging Self-Supervised Vision Transformers for Segmentation-Based Transfer Function Design., https://doi.org/10.1109/tvcg.2024.3401755, fetched 2026-10-26
