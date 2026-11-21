# X-Ray Volume Rendering

Also called: Average Volume Rendering, Linear Accumulation Volume Rendering, Ray-marching X-ray transform, Sparse matrix-vector product via ray tracing, Voxel-based X-ray Forward Projection, avgShader

## Overview
X-Ray Volume Rendering is a direct volume rendering technique that simulates a transmission view (such as a CT scan or radiograph) by linearly accumulating or averaging voxel intensity values along a ray, rather than using alpha compositing for opacity [S1, S2, unofficial]. Unlike standard ray casting which blends colors based on transparency, this method ignores alpha for the final accumulation logic, resulting in a transparent/transmissive appearance where the final color represents the mean or sum of densities encountered [S2, unofficial]. It differs from its sibling [Pixel Intensity Evaluation with Scene Aggregates](10890-pixel-intensity-evaluation-with-scene-aggregates.md) by operating on volumetric data via ray traversal rather than scene aggregates [S1, unofficial].

## How it works
The method approximates the X-ray projection of a voxelized function $\phi$ by summing the intersection lengths of rays with voxels weighted by voxel values, effectively computing the sparse matrix-vector product $X_C \phi$ without explicitly forming the matrix [S3].

### Inputs and Outputs
*   **Input**: A 3D volume texture (or grid) of size $n_1 \times n_2 \times n_3$, where each voxel $k$ has a value $\phi_k$ (typically RGBA, though only RGB is accumulated for color in average mode) [S1, unofficial][S3]. Camera parameters $C = (x_0, Q, K, m)$ defining the ray origin $x_0$ and direction $u$ [S3].
*   **Output**: A 2D image where each pixel $(i_1, i_2)$ contains the accumulated intensity $X[\phi](x_0, u)$ [S3].

### Data Structures
*   **Volume Grid**: Discretized parallelepiped $[a_1, b_1] \times [a_2, b_2] \times [a_3, b_3]$ into $N = n_1 \times n_2 \times n_3$ uniform voxels of side $h$ [S3].
*   **Basis Functions**: $\psi_k$ is the characteristic function of voxel $k$ (value 1 inside, 0 otherwise) [S3].
*   **Ray**: Defined by origin $x_0$ and direction $u$, $L(x_0, u) = \{ (x_1, x_2, x_3) : \lambda \ge 0 \}$ [S3].

### Parameters
*   `MAX_STEPS` (or `SAMPLES`): The number of discrete steps along the ray [S1, S2, unofficial].
*   `alphaCorrection`: A multiplier applied to the sampled alpha value [S1, unofficial].
*   `alphaScaleFactor`: A multiplier applied to alpha to make the result invariant to step size [S1, unofficial].
*   `rayLength`: The total length of the ray segment traversing the volume [S1, unofficial].

### Algorithm Steps
1.  **Ray-Volume Intersection**: Calculate the front intersection parameter $t_0$ and back intersection parameter $t_1$ of the ray with the volume's Axis-Aligned Bounding Box (AABB) [S1, S2, unofficial]. If no intersection, output background color.
2.  **Initialization**: Initialize `accumulatedColor` to $(0,0,0)$ and `accumulatedAlpha` to $0.0$ [S1, S2, unofficial].
3.  **Step Size Calculation**: Compute `sampleLength` as $(t_1 - t_0) / \text{SAMPLES}$ [S2, unofficial].
4.  **Traversal Loop**: Iterate $i$ from $0$ to $\text{SAMPLES}-1$ (or while `accumulatedLength` < `rayLength`) [S1, S2, unofficial]:
    a.  **Sampling**: Retrieve the interpolated color $c$ (RGBA) at the current ray position using trilinear interpolation or nearest neighbor [S1, S2, unofficial].
    b.  **Alpha Processing (Unused for Color)**:
        i.  Calculate `alphaSample` = $c.a \times \text{alphaCorrection}$ [S1, unofficial].
        ii. Modulate by remaining transparency: `alphaSample` *= $(1.0 - \text{accumulatedAlpha})$ [S1, unofficial].
        iii. Scale: `alphaSample` *= `alphaScaleFactor` [S1, unofficial].
        iv. Update `accumulatedAlpha` += `alphaSample` [S1, unofficial].
        *Note: This alpha logic is retained from standard ray casting but is not used in the final color output for X-Ray mode, potentially wasting computation [S1, unofficial].*
    c.  **Color Accumulation**:
        i.  Add the RGB components of $c$ to `accumulatedColor`. In the "Average" variant, divide by `MAX_STEPS` at each step or at the end [S1, S2, unofficial].
        ii. Formula: $\text{accumulatedColor} = \text{accumulatedColor} + \frac{c_{rgb}}{\text{MAX\_STEPS}}$ [S1, unofficial].
    d.  **Advance**: Update ray position by `deltaDirection` and `accumulatedLength` by `deltaDirectionLength` [S1, unofficial].
5.  **Termination**: Break if `accumulatedLength` >= `rayLength` or `accumulatedAlpha` >= $1.0$ [S1, unofficial].
6.  **Final Output**: If the ray hit the volume, divide the accumulated color by `SAMPLES` (if not done incrementally) to get the average [S2, unofficial]. Output `accumulatedColor` as the fragment color [S1, unofficial].

### Mathematical Formulation
The X-ray evaluation is defined as [S3]:
$$
X[\phi](x_0, u) = \sum_{k=1}^{N} \phi_k X[\psi_k](x_0, u)
$$
where $X[\psi_k](x_0, u)$ is the intersection length of the ray with voxel $k$ [S3]. In the discrete sampling approximation (Average Rendering), this is approximated by the mean of sampled values [S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def x_ray_rendering(ray_origin, ray_direction, volume_texture, max_steps, ray_length):
    # 1. Ray-AABB Intersection
    t0, t1 = intersect_aabb(ray_origin, ray_direction, volume_bounds)
    if t1 < t0:
        return background_color

    # 2. Initialization
    accumulated_color = vec3(0.0, 0.0, 0.0)
    accumulated_alpha = 0.0
    sample_length = (t1 - t0) / max_steps
    current_pos = ray_origin + ray_direction * t0
    
    # 3. Loop
    for i in range(max_steps):
        # Sample voxel
        color_sample = volume_texture.sample(current_pos) # Returns RGBA
        
        # Alpha processing (retained from standard ray casting, unused for final color)
        alpha_sample = color_sample.a * alpha_correction
        alpha_sample *= (1.0 - accumulated_alpha)
        alpha_sample *= alpha_scale_factor
        accumulated_alpha += alpha_sample
        
        # Color accumulation (Average X-Ray)
        accumulated_color += color_sample.rgb / max_steps
        
        # Advance
        current_pos += ray_direction * sample_length
        
        # Early exit check
        if accumulated_alpha >= 1.0:
            break

    # 4. Output
    return accumulated_color
```

### Edge Cases and Pitfalls
*   **Unused Alpha Computation**: The alpha accumulation logic (steps 3-4 and 7 in the source code) is retained from the original Ray Casting method but is not used in the final color output for X-Ray mode, potentially wasting computation [S1, unofficial].
*   **Precision**: Dividing by `MAX_STEPS` at each step or at the end requires careful handling of floating-point precision to avoid accumulation errors over large volumes [S1, unofficial].
*   **Boundary Conditions**: If the ray does not intersect the volume, the method must handle the null case explicitly [S1, unofficial].

### Subtopics
*   **Linear Accumulation**: Sums voxel values directly. Chosen when density is additive and opacity is ignored [S1, unofficial].
*   **Average Volume Rendering**: Divides the sum by the number of steps. Chosen for a normalized "transmission" look [S2, unofficial].
*   **Sparse Matrix-Vector Product**: The mathematical abstraction of this rendering, treating the volume as a sparse matrix $X_C$ [S3].

## Operating characteristics
*   **Time Complexity**: $O(k_n k)$, where $k_n$ is the number of crossed voxels and $k$ is a constant factor related to the number of diagonal voxels ($\sqrt{n_1^2 + n_2^2 + n_3^2}$ is an upper bound) [S3]. In discrete sampling, it is $O(\text{SAMPLES})$ per pixel [S1, unofficial].
*   **Memory Complexity**: $O(N)$ for the volume data, where $N = n_1 \times n_2 \times n_3$ [S3].
*   **Performance**: Efficient for large volumes as it avoids explicit matrix formation [S3].
*   **Strengths**: Simulates X-ray/CT appearance; computationally simpler than alpha compositing [S1, S2, unofficial].
*   **Limitations**: Ignores opacity/transparency effects; alpha calculation is wasted if not used [S1, unofficial].

## Background
Originating from the need to efficiently compute the X-ray projection of voxelized functions without forming huge sparse matrices, this method is based on the work of [S3] (Flexible algebraic technique for multiview reconstruction). It is also known as "Average Volume Rendering" in implementations like [S2, unofficial] and is a modification of standard ray casting code [S1, unofficial].

## Sources
[S1] Convert from Simple Ray Casting to X-Ray Rendering, https://stackoverflow.com/questions/48757096/convert-from-simple-ray-casting-to-x-ray-rendering, fetched 2026-10-26 (unofficial)
[S2] CSE-333-Computer-Graphics-2020/VolumeRayCasting, https://github.com/CSE-333-Computer-Graphics-2020/VolumeRayCasting, fetched 2026-10-26 (unofficial)
[S3] Flexible algebraic technique for multiview reconstruction: incremental learning in reflective tomography, https://doi.org/10.1117/1.OE.58.10.103102, fetched 2026-10-26
