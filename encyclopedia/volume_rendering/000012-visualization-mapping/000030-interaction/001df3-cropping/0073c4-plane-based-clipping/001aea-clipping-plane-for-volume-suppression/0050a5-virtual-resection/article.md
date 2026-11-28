# Virtual Resection

Also called: Deformable Clipping Plane, Resection via Clipping Plane, Selective Volume Clipping, Selective Volume Clipping in Resection of Lung Cancer

## Overview
Virtual Resection is a technique for selectively suppressing portions of a volume dataset to resolve occlusion problems, particularly in medical imaging where internal structures (e.g., pulmonary nodules) are hidden by surrounding tissue (e.g., lung parenchyma) [S2]. It generates a deformable clipping plane or mask from user-defined resection lines on the organ's surface, allowing the user to cut away specific regions while preserving the texture of the remaining surface [S1, unofficial][S2]. Unlike standard clipping planes that cut the entire volume uniformly, this method enables complex, arbitrary-shaped cutaways centered on regions of interest [S2].

## How it works
The method operates in two distinct contexts found in the literature: a general deformable plane manipulation for anatomical models [S1, unofficial] and a specific computational framework for lung nodule visualization [S2]. The implementation details below describe the [S2] framework, as it provides a complete algorithmic pipeline.

### Inputs and Outputs
*   **Input:** A 3D volume dataset $V$ (e.g., chest CT images) containing scalar values $f(x, y, z)$ [S2].
*   **Output:** A volume rendering where voxels inside a user-defined closed curve (but outside the nodule) are suppressed, revealing internal structures [S2].

### Data Structures
1.  **Feature Space Histogram:** A 2D histogram where the horizontal axis represents gray scale intensity and the vertical axis represents gradient magnitude $\nabla f(x, y, z)$ [S2].
2.  **Voxel Set $\Phi_U$:** A set of voxels selected from the volume based on a region $U$ in the feature space [S2].
3.  **Stack $H$:** A stack data structure used for connected component tracing [S2].
4.  **Binary Mask Image:** A 2D image with the same dimensions as the screen rendering window, representing the inside/outside status of the closed curve [S2].

### Algorithm Steps
The process involves segmentation, mask generation, and clipping.

1.  **Segment Lungs via 2D Feature Space:**
    *   Define the 2D feature space with coordinates $(g, \nabla g)$, where $g$ is gray scale and $\nabla g$ is gradient magnitude [S2].
    *   Identify region $R_1$ as homogeneous regions with low gray gradient, corresponding to lung tissue [S2].
    *   Define the selection region $U = R_1$.
    *   Determine the initial voxel set $\Phi_U$ using the condition that voxels satisfy the gray scale and gradient thresholds for $U$ [S2].

2.  **Extract Connected Lung Volume:**
    *   Mark all voxels in $\Phi_U$ as `not-visited` [S2].
    *   Select a `not-visited` voxel from $\Phi_U$ as a seed. Mark it `visited` and push it onto stack $H$ [S2].
    *   While $H$ is not empty:
        1.  Pop a voxel $v$ from $H$ [S2].
        2.  Check all voxels in the 18-neighborhood of $v$ [S2].
        3.  For each neighbor $n$:
            *   If $n \notin \Phi_U$ or $n$ is already `visited`, skip [S2].
            *   If $n \in \Phi_U$ and `not-visited`, push $n$ to $H$ and mark $n$ as `visited` [S2].
    *   The result is the extracted connected lung volume [S2].

3.  **Segment Pulmonary Nodules:**
    *   Apply a region growing method to segment nodules from the lung tissue [S2].

4.  **Generate Deformable Clipping Mask:**
    *   Project voxels from the volume coordinate system onto the screen frame [S2].
    *   The user draws discontinuous points on the screen around the region of interest (e.g., the nodule) [S2].
    *   Fit these points using a B-Spline to generate a closed curve [S2].
    *   Create a binary mask image of the same size as the screen window. Voxels inside the closed curve are marked as `inside`, and those outside are marked as `outside` [S2].

5.  **Selective Clipping:**
    *   For each voxel in the volume:
        *   Project the voxel to the screen.
        *   If the voxel is inside the closed curve AND is part of the lung tissue (not the nodule), suppress it (clip it) [S2].
        *   Otherwise, retain it for rendering [S2].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def virtual_resection(volume, screen_width, screen_height):
    # Step 1: Segment Lungs via 2D Feature Space
    # Horizontal axis: gray scale, vertical axis: gradient magnitude [S2]
    # Region R1: homogeneous low gray gradient (lungs) [S2]
    # Voxel set Phi_U determined by U = R1 [S2]
    Phi_U = set()
    for x in range(volume.shape[0]):
        for y in range(volume.shape[1]):
            for z in range(volume.shape[2]):
                gray = volume[x, y, z]
                grad = compute_gradient_magnitude(volume, x, y, z)
                if gray < threshold_gray and grad < threshold_grad:  # [S2] R1 definition
                    Phi_U.add((x, y, z))

    # Step 2: Extract Connected Lung Volume
    visited = {v: False for v in Phi_U}
    H = []
    seed = next(iter(Phi_U))  # [G] Standard initialization
    visited[seed] = True
    H.append(seed)
    while H:
        v = H.pop()
        for n in get_18_neighbors(v):
            if n in Phi_U and not visited[n]:
                visited[n] = True
                H.append(n)
    connected_lung = set(visited.keys())

    # Step 3: Segment Pulmonary Nodules
    nodules = set()  # [S2] Region growing method not detailed in sources
    # Apply region growing to separate nodules from lung tissue [S2]

    # Step 4: Generate Deformable Clipping Mask
    # Project volume voxels to screen frame [S2]
    # User draws discontinuous points on screen ROI [S2]
    user_points = []  # [G] Standard UI interaction
    # Fit B-Spline to generate closed curve [S2]
    closed_curve = []  # [G] Standard spline fitting
    # Create binary mask image same size as screen window [S2]
    binary_mask = create_binary_mask(closed_curve, screen_width, screen_height)  # [G] Standard rasterization

    # Step 5: Selective Clipping
    rendered_volume = []
    for v in volume:
        sx, sy = project_to_screen(v)  # [G] Standard projection
        inside = binary_mask[sy][sx]
        is_lung = v in connected_lung
        is_nodule = v in nodules
        if inside and is_lung and not is_nodule:
            continue  # Suppress voxel [S2]
        else:
            rendered_volume.append(v)  # Retain for rendering [S2]
    return rendered_volume

def compute_gradient_magnitude(volume, x, y, z):
    # Standard central difference approximation [G]
    dx = (volume[x+1, y, z] - volume[x-1, y, z]) / 2.0
    dy = (volume[x, y+1, z] - volume[x, y-1, z]) / 2.0
    dz = (volume[x, y, z+1] - volume[x, y, z-1]) / 2.0
    return (dx**2 + dy**2 + dz**2)**0.5

def get_18_neighbors(v):
    # Returns 18-connected neighbors in 3D grid [S2]
    return [(v[0]+dx, v[1]+dy, v[2]+dz) for dx in [-1,0,1] for dy in [-1,0,1] for dz in [-1,0,1] if not (dx==0 and dy==0 and dz==0)]

def create_binary_mask(curve, width, height):
    # Standard point-in-polygon rasterization [G]
    mask = [[False] * width for _ in range(height)]
    # Rasterize closed_curve into mask [G]
    return mask

def project_to_screen(v):
    # Standard volume-to-screen projection [G]
    return (v[0], v[1])
```

### Parameters and Edge Cases
*   **18-Neighbor Connectivity:** The tracing algorithm uses 18-connectivity, which is specific to 3D voxel grids [S2].
*   **Occlusion:** The primary edge case addressed is the occlusion of nodules by lung tissue, which semi-transparent visualization fails to resolve due to loss of surface texture [S2].
*   **B-Spline Fit:** The sources do not specify the minimum number of points required for stability [S2].

## Operating characteristics
*   **Strengths:** Solves the occlusion problem in chest CT images while preserving the texture of the pulmonary tongue on the lung surface, which is lost in semi-transparent visualization [S2]. Allows selective cutting layer by layer, excluding internal parts from being cut [S1, unofficial].
*   **Limitations:** Requires initial segmentation of lungs and nodules, which adds computational overhead [S2]. The deformable plane is manipulated via mouse interaction, which may be less precise than parametric clipping planes for complex geometries [S1, unofficial].
*   **Performance:** Not given in the sources.
*   **When to use:** When visualizing internal structures (like nodules) hidden within a larger organ (like lungs) where surface texture preservation is critical [S2].

## Background
Virtual Resection was introduced by Konrad-Verse et al. [S1, unofficial]. It is also referred to as Selective Volume Clipping in the context of lung cancer resection visualization [S2]. The method addresses the limitations of standard clipping planes and semi-transparent rendering in medical visualization [S1, unofficial][S2].

## Sources
[S1] Enhanced perception in volume visualization, https://doi.org/10.5821/dissertation-2117-94929, fetched 2026-10-26 (unofficial)
[S2] Selective Volume Clipping in Resection of Lung Cancer, https://doi.org/10.1145/3143344.3143358, fetched 2026-11-05
