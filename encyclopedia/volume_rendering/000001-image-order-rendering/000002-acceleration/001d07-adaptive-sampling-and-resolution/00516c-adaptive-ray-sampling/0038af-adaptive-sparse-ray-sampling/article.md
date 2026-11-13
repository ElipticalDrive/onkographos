# Adaptive Sparse Ray Sampling

Also called: Sparse Grid Ray Sampling, Adaptive Image Subdivision for Volume Rendering

## Overview
Adaptive Sparse Ray Sampling is an image-order acceleration technique that reduces computational cost by casting a sparse grid of rays (density < 1 ray per pixel) and adaptively increasing sampling density in regions of high image complexity [S1, unofficial]. It exploits coherence in projections, particularly in data acquired from sensing devices where blurring introduces spatial correlation [S1, unofficial]. Unlike brute-force rendering, it achieves savings of more than two orders of magnitude by deferring full-resolution sampling to areas requiring detail [S1, unofficial].

## How it works
The method operates in an iterative refinement loop, generating successively more refined images over time [S1, unofficial].

**Inputs:**
*   Volume dataset $V$.
*   Camera parameters (view matrix, projection matrix, viewport dimensions $W \times H$).
*   Initial sampling density $\rho_0 < 1$ (e.g., 1 ray per $2 \times 2$ pixel block).
*   Threshold $\tau$ for local complexity (color difference).

**Outputs:**
*   Final image $I$ at display resolution.

**Data Structures:**
*   **Sample Array:** A list of tuples $(u, v, c)$, where $(u, v)$ are normalized screen coordinates and $c$ is the computed color (transmittance and color accumulation) for that ray.
*   **Image Buffer:** A $W \times H$ texture initialized to empty or previous iteration's result.

**Steps:**
1.  **Initial Sparse Casting:**
    Cast rays at a sparse grid of screen positions. For a density $\rho_0 = 1/k^2$, cast rays at coordinates $(i \cdot k, j \cdot k)$ for $i \in [0, W/k), j \in [0, H/k)$.
    For each ray, perform volume rendering (e.g., front-to-back integration) to compute color $c$.
    Store $(u, v, c)$ in the Sample Array.

2.  **Complexity Measurement:**
    Divide the image space into blocks (e.g., $2 \times 2$ pixels). For each block, compute the local complexity metric $M$.
    $M$ is defined as the maximum color difference between adjacent samples in the Sample Array that fall within or border the block.
    $$ M_{block} = \max_{(u_i, v_i), (u_j, v_j) \in \text{block}} \| c_i - c_j \| $$
    If a block contains only one sample, $M_{block} = 0$.

3.  **Adaptive Subdivision:**
    Identify blocks where $M_{block} > \tau$.
    Recursively subdivide these blocks into smaller regions (e.g., halving the block size).
    For each new sub-region, cast new rays at the center or corners if they are not already present in the Sample Array.
    Perform volume rendering for new rays and add $(u, v, c)$ to the Sample Array.
    Repeat subdivision until $M_{block} \le \tau$ for all blocks or a maximum depth is reached.

4.  **Interpolation and Resampling:**
    Construct the final image $I$ by interpolating colors from the nonuniform Sample Array to the regular $W \times H$ display grid.
    Use bilinear or bicubic interpolation based on the nearest neighbors in the Sample Array.
    $$ I(u, v) = \text{Interpolate}(\{(u_k, v_k, c_k)\}, u, v) $$

5.  **Iterative Refinement (Optional):**
    To generate high-quality images over time, repeat steps 2-4 with a lower threshold $\tau$ or higher initial density, adding new rays to the existing Sample Array and re-interpolating.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_sparse_ray_sampling(volume, camera, width, height, initial_density, threshold, max_depth):
    # Step 1: Initial sparse casting
    sample_array = []
    step = int(1.0 / sqrt(initial_density))
    for i in range(0, width, step):
        for j in range(0, height, step):
            ray = camera.cast_ray(i, j)
            color = volume_render(volume, ray)
            sample_array.append((i, j, color))

    # Step 2 & 3: Recursive subdivision
    def subdivide(samples, x, y, w, h, depth):
        if depth >= max_depth:
            return
        
        # Calculate complexity metric for this block
        # Find samples within or near the block [x, x+w] x [y, y+h]
        block_samples = [s for s in samples if x <= s[0] < x+w and y <= s[1] < y+h]
        
        max_diff = 0.0
        if len(block_samples) > 1:
            for i in range(len(block_samples)):
                for j in range(i + 1, len(block_samples)):
                    diff = norm(block_samples[i][2] - block_samples[j][2])
                    if diff > max_diff:
                        max_diff = diff
        
        if max_diff > threshold:
            # Subdivide into 4 quadrants
            half_w, half_h = w / 2, h / 2
            subdivide(samples, x, y, half_w, half_h, depth + 1)
            subdivide(samples, x + half_w, y, half_w, half_h, depth + 1)
            subdivide(samples, x, y + half_h, half_w, half_h, depth + 1)
            subdivide(samples, x + half_w, y + half_h, half_w, half_h, depth + 1)
            
            # Cast new rays at new centers if not present
            centers = [
                (x + half_w/2, y + half_h/2),
                (x + half_w + half_w/2, y + half_h/2),
                (x + half_w/2, y + half_h + half_h/2),
                (x + half_w + half_w/2, y + half_h + half_h/2)
            ]
            for cx, cy in centers:
                # Check if already in sample_array
                if not any(s[0] == cx and s[1] == cy for s in samples):
                    ray = camera.cast_ray(cx, cy)
                    color = volume_render(volume, ray)
                    samples.append((cx, cy, color))
        else:
            # Block is smooth, no further subdivision needed
            pass

    # Initial block size
    block_size = width * initial_density # Approximate block size in pixels
    subdivide(sample_array, 0, 0, width, height, 0)

    # Step 4: Interpolation
    image = create_empty_image(width, height)
    for u in range(width):
        for v in range(height):
            image[u, v] = interpolate(sample_array, u, v)
            
    return image
```

**Edge Cases and Pitfalls:**
*   **Precision:** Interpolation artifacts may occur if the sample density is too low in high-frequency regions before subdivision. [G] Ensure interpolation kernel supports non-uniform grids.
*   **Boundaries:** Handle rays that miss the volume or hit empty space efficiently by skipping integration. [S2, unofficial]
*   **Degenerate Input:** If the volume is homogeneous, the complexity metric will be low, and the algorithm will maintain sparse sampling, providing maximum speedup. [S1, unofficial]

## Subtopics

### Adaptive Image Supersampling
An acceleration method exploiting pixel-space coherency by casting rays from a subset of screen pixels and interpolating empty pixels between them [S2, unofficial]. It casts rays from only a subset of screen pixels (e.g., every other pixel) [S2, unofficial]. Empty pixels residing between pixels with similar values are assigned interpolated values [S2, unofficial]. In areas of high image gradient, additional rays are cast to resolve ambiguities [S2, unofficial]. This method was originally developed for traditional ray-tracing and later adapted to volume rendering [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \log N)$ or better, where $N$ is the number of pixels, due to adaptive subdivision. The number of rays cast is significantly less than $W \times H$ for smooth regions.
*   **Memory Complexity:** $O(K)$, where $K$ is the number of rays cast, which is much smaller than $W \times H$ in sparse regions.
*   **Performance:** Savings of more than two orders of magnitude over brute-force rendering methods were obtained for many datasets [S1, unofficial]. Crude images can be obtained in a few seconds, with gradually better images at intervals of a few seconds each, culminating in high-quality images in less than a minute [S1, unofficial].
*   **Strengths:** Significant speedup for coherent data; allows progressive refinement.
*   **Limitations:** Interpolation artifacts if subdivision is insufficient; overhead of managing non-uniform sample arrays.
*   **When to use:** When rendering coherent volumetric data (e.g., medical scans, cloud simulations) where large regions are smooth.
*   **When not to use:** For highly incoherent or noisy data where subdivision depth becomes excessive.

## Background
The method was introduced in the context of interactive visualization of 3D medical data [S1, unofficial]. It builds on classical ray tracing techniques for non-uniform ray distribution, such as recursive subdivision of image space [S1, unofficial]. The approach is similar to Whitted's method but extended to allow sampling densities of less than one ray per pixel [S1, unofficial]. It exploits the coherence present in projections of data acquired from sensing devices, where blurring introduces spatial correlation [S1, unofficial].

## Sources
[S1] DTIC ADA208091: Interactive Visualization of 3D Medical Data, https://archive.org/details/DTIC_ADA208091, fetched 2026-10-26 (unofficial)
[S2] Volume Visualization and Volume Rendering Techniques, https://www.cs.unh.edu/~cs880/visualization/Meissner-VolRenderingEGTutorial-rdb.pdf, fetched 2026-10-27 (unofficial)
