# Maximum Intensity Projection (MIP)

Also called: MIP Shader, Max intensity

## Overview
Maximum Intensity Projection (MIP) is a volume rendering technique that projects the maximum scalar value encountered along each viewing ray onto the 2D image plane [S1, unofficial]. It is primarily used to highlight high-intensity structures, such as blood vessels in angiography or bone in CT scans, by suppressing lower-intensity background noise [S1, unofficial]. Unlike standard ray casting which composites opacity and color, MIP is order-independent and discards depth information, meaning it loses the ability to differentiate between front and back structures that share the same maximum intensity [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar volume dataset $V$, a camera position $\mathbf{o}$, a viewing direction $\mathbf{d}$, and a step size $\Delta t$ (or number of steps $N$).
*   **Output:** A single scalar value $A$ per pixel, which is subsequently mapped to a color $C$ via a transfer function.

**Data Structures:**
*   **Volume:** A 3D texture or array storing scalar values. In the reference implementation, this is a `Data3DTexture` with `RedFormat` and `FloatType` [S2, unofficial].
*   **Ray:** Defined by origin $\mathbf{o}$ and direction $\mathbf{d}$, intersecting the volume bounding box [Parent].

**Algorithm Steps:**
1.  **Ray Initialization:** For each pixel, construct the ray $r(t) = \mathbf{o} + t\mathbf{d}$ [Parent].
2.  **Intersection:** Determine the entry and exit points of the ray with the volume bounding box [Parent].
3.  **Discretization:** Divide the ray segment within the volume into $N$ samples with step size $\Delta t$ [S2, unofficial].
4.  **Accumulation:** Initialize the accumulator $A$ to the minimum representable value (identity for max) [Parent]. Iterate through each sample point $t_i$ along the ray:
    *   Sample the scalar value $s_i$ from the volume using trilinear interpolation [Parent].
    *   Update the accumulator: $A \leftarrow \max(A, s_i)$ [S1, S2, unofficial][Parent].
5.  **Classification:** Map the final maximum value $A$ to a color $C$ using a transfer function $T(A)$ [Parent].
    *   Formula: $C(R) = T(\max_{i} s_i)$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_mip(ray_origin, ray_direction, volume, volume_dims, num_steps):
    # 1. Intersect ray with volume bounding box
    # [G] Assuming axis-aligned bounding box [0, width] x [0, height] x [0, depth]
    # Calculate t_min and t_max for entry and exit
    t_min, t_max = intersect_box(ray_origin, ray_direction, volume_dims)
    
    if t_max < t_min:
        return None  # Ray misses volume

    # 2. Discretize ray
    step_size = (t_max - t_min) / num_steps
    current_t = t_min
    
    # 3. Initialize accumulator with minimum float value
    max_val = -float('inf')

    for i in range(num_steps):
        # 4. Sample volume
        sample_point = ray_origin + ray_direction * current_t
        
        # [G] Trilinear interpolation from volume at sample_point
        # Returns scalar value s_i
        s_i = trilinear_interpolate(volume, sample_point, volume_dims)
        
        # 5. Update max accumulator
        if s_i > max_val:
            max_val = s_i
            
        current_t += step_size

    # 6. Return max value for color mapping
    return max_val
```

**Parameters:**
*   **Steps ($N$):** The number of samples along the ray. A higher $N$ increases accuracy but also computational cost [S2, unofficial]. Typical values depend on resolution; the reference code uses 200 steps [S2, unofficial].
*   **Step Size ($\Delta t$):** Often derived from $N$ and the ray length. Regular sampling is common, though it may miss details if the step size is too large relative to high-frequency features [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Loss of Depth:** MIP does not preserve depth cues. If two structures (one in front, one behind) have the same maximum intensity, they appear identical, and their relative positions are lost [S1, unofficial].
*   **Noise Sensitivity:** MIP is sensitive to noise. High-intensity noise artifacts can obscure true anatomical structures [S1, unofficial].
*   **Boundary Conditions:** Incorrect ray-box intersection or out-of-bounds sampling can lead to artifacts. Proper clamping or filtering is required [S1, unofficial].

## Subtopics

### Minimum Intensity Projection (MinIP)
Projects the minimum intensity along the ray. Useful for visualizing low-density structures like air-filled lungs. [Link](5299-minimum-intensity-projection.md)

### Projected Slabs Approximation
Approximates MIP by projecting a slab of volume data, offering a balance between speed and accuracy. [Link](1449-projected-slabs-approximation.md)

### Forward Projection (aTx)
A different aggregation method that sums values, often used in tomography. [Link](1492-forward-projection-atx.md)

## Operating characteristics
*   **Time Complexity:** $O(N)$ per pixel, where $N$ is the number of samples along the ray. Total complexity is $O(W \cdot H \cdot N)$ for an image of size $W \times H$ [Parent].
*   **Memory Complexity:** $O(V)$ for the volume data, where $V$ is the number of voxels [Parent].
*   **Strengths:** Simple to implement; highly effective for highlighting high-contrast structures; order-independent, allowing for flexible traversal and early termination if a maximum threshold is reached [Parent].
*   **Limitations:** Loses depth information; sensitive to noise; does not provide a realistic 3D appearance due to lack of shading and opacity [S1, unofficial].
*   **When to Use:** Angiography, bone visualization, or any scenario where high-intensity features are of primary interest and depth is secondary.
*   **When Not to Use:** When depth perception is critical, or when the volume contains significant noise that could be mistaken for signal.

## Background
MIP is a variation of direct volume rendering that projects the maximum intensity along the ray [S1, unofficial]. It is widely used in medical imaging, particularly in CT and MRI, for visualizing vascular structures and bones. The technique is computationally efficient and has been implemented in various graphics APIs and libraries, including custom shaders in WebGL environments [S2, unofficial].

## Sources
[S1] Direct Volume Rendering Direct Volume Rendering. What is volume rendering?. Accumulate information along 1 dimension line through volume. Volume rendering vs. isosurfaces. No intermediate geometry No , https://www.slideserve.com/lenci/direct-volume-rendering, fetched 2026-10-25 (unofficial)
[S2] niklasgruener/volume_vis, https://github.com/niklasgruener/volume_vis, fetched 2026-10-26 (unofficial)
