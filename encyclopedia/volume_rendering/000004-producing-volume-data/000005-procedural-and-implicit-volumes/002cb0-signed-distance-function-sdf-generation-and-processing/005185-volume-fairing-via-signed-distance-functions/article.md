# Volume Fairing via Signed Distance Functions

Also called: ESDF interpolation, linear combination of SDFs

## Overview
Volume Fairing via Signed Distance Functions is a preprocessing technique used to smooth material boundaries in segmented volumes, particularly when slices are misaligned or recorded at different times (e.g., different heart cycles in medical imaging) [S1]. It solves the problem of artifacts caused by slice gaps and misalignment by interpolating between slices and fairing boundaries using Euclidean Signed Distance Functions (ESDFs) [S1]. Unlike methods that compute 3D SDFs for the entire volume, this approach computes ESDFs only on individual 2D slices for efficiency, allowing for the interpolation and fairing of multiple material boundaries simultaneously [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 2D binary or multi-material slices $I_i$ representing segmented volumes. Parameters include the fairing constant $\alpha$ and the maximum expected feature distance $d_{max}$ [S1].
*   **Output:** A smoothed (fairing) or interpolated volume with continuous material boundaries across slices [S1].

**Data Structures:**
*   **ESDF Map ($s_{m,i}$):** A 2D grid of floating-point values for each material $m$ in slice $i$. Values represent the Euclidean distance to the material boundary.
    *   **Layout:** $s_{m,i}(x, y)$ stores the signed distance.
    *   **Sign Convention:** Negative values indicate the interior of the material; positive values indicate the exterior [S1].
    *   **Clamping:** Values are bounded within $[-d_{max}, d_{max}]$ to handle materials not present in all slices [S1].

**Algorithm Steps:**

1.  **ESDF Computation per Slice:**
    For each slice $i$ and each material $m$:
    a.  Identify boundary pixels where the material transitions from $m$ to non-$m$. Initialize these pixels in $s_{m,i}$ to $0$ [S1].
    b.  Propagate the distance front iteratively from the boundary. Fill adjacent pixels with distances $1, \sqrt{2}, 2, \sqrt{5}, \dots$ based on their connectivity (4-connected or 8-connected Euclidean distance) [S1].
    c.  Flip the sign of the distance values inside the feature domain (interior) to create a signed map: negative inside, positive outside [S1].
    d.  Clamp all values to $[-d_{max}, d_{max}]$ [S1].

2.  **Interpolation (Between two slices $0$ and $1$):**
    For a parameter $t \in [0, 1]$:
    a.  Compute the linear combination of ESDFs:
        $$ s_t(x, y) = (1 - t) s_0(x, y) + t s_1(x, y) $$
    b.  Assign material to pixels where $s_t(x, y) < 0$ [S1].

3.  **Fairing (Smoothing a middle slice $i$):**
    Using adjacent slices $i-1$ and $i+1$:
    a.  Compute the fairing update for each material $m$:
        $$ s_{m,i}^{new} = (1 - 2\alpha) s_{m,i} + \alpha (s_{m,i-1} + s_{m,i+1}) $$
    b.  Re-apply the sign convention: The material is assigned to pixels where the resulting combined distance is negative. Note that the formula above applies to the signed distance values directly. The resulting slice is determined by the material with the minimal distance (most negative) at each pixel if multiple materials are present [S1].
    c.  The fairing process shifts the material boundary by at most one slice distance $\epsilon_s$ [S1].

4.  **Multiple Materials Handling:**
    a.  Construct an ESDF $s_{m,i}$ for every material $m$ in every participating slice $i$ [S1].
    b.  Perform convex combinations (interpolation or fairing) for each material independently [S1].
    c.  For each pixel in the resulting slice, assign the material corresponding to the minimal distance value among all materials [S1].

5.  **Topological Ambiguity Resolution (Optional):**
    If topological information is available (e.g., joining vessels):
    a.  Treat connected components composed of multiple materials as a single "dummy" material in a first pass to eliminate feature lines between them [S1].
    b.  Separate the participating materials in a second pass [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_esdf(slice_data, material_id, d_max):
    """
    Computes the Euclidean Signed Distance Function for a specific material in a 2D slice.
    slice_data: 2D array of material IDs.
    material_id: The ID of the material to compute ESDF for.
    d_max: Maximum distance to clamp.
    Returns: 2D array of signed distances.
    """
    h, w = slice_data.shape
    esdf = np.full((h, w), np.inf)
    
    # Step 1: Initialize boundary pixels to 0
    # A pixel is on the boundary if it is the material and has a neighbor that is not.
    boundary_mask = (slice_data == material_id) & \
                    ((slice_data[1:, :] != material_id) | (slice_data[:-1, :] != material_id) |
                     (slice_data[:, 1:] != material_id) | (slice_data[:, :-1] != material_id))
    esdf[boundary_mask] = 0.0
    
    # Step 2: Propagate distance iteratively (BFS-like or iterative relaxation)
    # Using a simple iterative approach for clarity, though BFS is more efficient.
    # In practice, use a fast marching method or BFS for exact Euclidean distances.
    changed = True
    while changed:
        changed = False
        # Check 4-neighbors and diagonal neighbors for distance updates
        # This is a simplified representation of the propagation described in [S1]
        # Real implementation requires exact Euclidean distance calculation (1, sqrt(2), etc.)
        for y in range(h):
            for x in range(w):
                if esdf[y, x] == 0.0:
                    # Update neighbors
                    for dy, dx in [(-1,0), (1,0), (0,-1), (0,1), (-1,-1), (-1,1), (1,-1), (1,1)]:
                        ny, nx = y + dy, x + dx
                        if 0 <= ny < h and 0 <= nx < w:
                            if esdf[ny, nx] == np.inf:
                                dist = np.sqrt(dy**2 + dx**2)
                                esdf[ny, nx] = dist
                                changed = True
                                # Note: The iterative propagation in [S1] fills values incrementally.
                                # A single pass is insufficient for exact Euclidean distances.
                                # A proper implementation would use a queue-based BFS or Fast Marching.
    
    # Step 3: Flip sign inside the feature domain
    # Interior pixels are those where slice_data == material_id and esdf was originally 0 or propagated from 0
    # However, the text says "flip the sign inside the feature domain".
    # Since we initialized boundary to 0 and propagated outwards, the interior is where the original mask was true.
    interior_mask = (slice_data == material_id)
    esdf[interior_mask] = -esdf[interior_mask]
    
    # Step 4: Clamp to [-d_max, d_max]
    esdf = np.clip(esdf, -d_max, d_max)
    
    return esdf

def fairing_slice(esdf_prev, esdf_curr, esdf_next, alpha):
    """
    Applies fairing to the current slice's ESDF using adjacent slices.
    """
    # Formula: s_i = (1 - 2*alpha) * s_i + alpha * (s_{i-1} + s_{i+1})
    return (1 - 2 * alpha) * esdf_curr + alpha * (esdf_prev + esdf_next)

def interpolate_slices(esdf_0, esdf_1, t):
    """
    Interpolates between two slices' ESDFs.
    """
    return (1 - t) * esdf_0 + t * esdf_1

def assign_material_from_esdf(esdf_map, materials):
    """
    Assigns material to each pixel based on the minimal distance.
    esdf_map: Dictionary mapping material_id -> 2D ESDF array.
    Returns: 2D array of material IDs.
    """
    h, w = esdf_map[materials[0]].shape
    result = np.zeros((h, w), dtype=int)
    
    for y in range(h):
        for x in range(w):
            min_dist = np.inf
            best_material = None
            for m in materials:
                dist = esdf_map[m][y, x]
                if dist < min_dist:
                    min_dist = dist
                    best_material = m
            result[y, x] = best_material
            
    return result
```

**Parameters:**
*   $\alpha$: Fairing constant. Typical value: $0.1$. Range: $[0, 1/3]$ [S1].
*   $d_{max}$: Maximal expected distance of features in adjacent slices. Used to bound ESDF range to $[-d_{max}, d_{max}]$ [S1].

**Edge Cases and Pitfalls:**
*   **Infinite Values:** If a material is not present in all participating images, ESDF values can become infinite. Clamping to $[-d_{max}, d_{max}]$ avoids this [S1].
*   **Topological Ambiguities:** Features that should be connected (e.g., joining vessels) may appear disconnected due to slice misalignment. Treating multi-material connected components as a single "dummy" material in a first pass eliminates feature lines between them [S1].
*   **Boundary Shift:** The fairing process shifts the material boundary by at most one slice distance $\epsilon_s$ [S1].

## Operating characteristics
*   **Time Complexity:** Depends on the ESDF computation method. The iterative propagation described is $O(N)$ per slice if using BFS/Fast Marching, where $N$ is the number of pixels. Fairing is $O(N)$ per slice.
*   **Memory Complexity:** $O(N \cdot M)$, where $N$ is the number of pixels per slice and $M$ is the number of materials.
*   **Strengths:** Efficient compared to 3D SDF computation; handles multiple materials; reduces artifacts from slice misalignment [S1].
*   **Limitations:** Limited to 2D slice processing; may not capture complex 3D topological features without the dummy material pass; boundary shift is limited to one slice distance [S1].
*   **When to use:** When dealing with segmented volumes from non-simultaneous slices (e.g., cardiac MRI) where misalignment causes artifacts [S1].
*   **When not to use:** When high-fidelity 3D topology is critical and slice-based smoothing is insufficient [S1].

## Background
The method was introduced by Bertram et al. in 2005 as part of "Non-manifold Mesh Extraction from Time-varying Segmented Volumes used for Modeling a Human Heart" [S1]. It builds on the work of Jones/Chen [JC94], who used 3D SDFs for reconstructing material boundaries from contours, but adapts it for efficiency by computing ESDFs only on individual slices [S1]. The technique is particularly relevant in medical imaging, such as modeling the human heart, where slices are often acquired over different heart cycles [S1].

## Sources
[S1] Non-manifold Mesh Extraction from Time-varying Segmented Volumes used for Modeling a Human Heart, https://doi.org/10.2312/vissym/eurovis05/199-206, fetched 2026-10-26
