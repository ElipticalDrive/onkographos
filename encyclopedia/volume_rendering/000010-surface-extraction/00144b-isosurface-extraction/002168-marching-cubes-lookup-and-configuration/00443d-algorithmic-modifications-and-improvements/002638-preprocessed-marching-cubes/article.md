# Preprocessed Marching Cubes

Also called: Marching Cubes with preprocessing, meshify, niimath meshify

## Overview
Preprocessed Marching Cubes extracts a triangle mesh from a 3D scalar volume by applying optional preprocessing steps—such as Gaussian smoothing, connected-component filtering, and boundary darkening—before running the standard Marching Cubes algorithm [S1, unofficial]. It sets itself apart from standard Marching Cubes by actively isolating the largest object, filling internal voids, and preventing boundary voxels from being missed due to cropping artifacts [S1, unofficial]. Unlike [Snap Marching Cubes](2181-snap-marching-cubes.md) which modifies the scalar field values to snap to the isovalue, or [Bitwise Marching Cubes](7374-bitwise-marching-cubes.md) which relies on bitwise operations for configuration, this method focuses on topological and geometric cleanup of the input volume to ensure robust surface extraction [S1, unofficial].

## How it works
**Inputs:**
- `img`: A 1D array of $N_{vox} = N_X \times N_Y \times N_Z$ floating-point values representing the scalar volume.
- `dim`: A 3D vector $(N_X, N_Y, N_Z)$ representing the grid dimensions.
- `isolevel`: The scalar threshold for surface extraction.
- `preSmooth`: Boolean flag to apply Gaussian blur.
- `onlyLargest`: Boolean flag to extract only the largest connected component.
- `fillBubbles`: Boolean flag to fill internal voids within the largest component.

**Outputs:**
- `pts`: An array of vertices (struct `vec3s`).
- `tris`: An array of triangle indices.
- `npt`: Number of vertices.
- `ntri`: Number of triangles.

**Data Structures:**
- `vec3s`: A struct containing three `float` fields `x`, `y`, `z` (12 bytes total) [S1, unofficial].
- `mask`: A 1D boolean array of size $N_{vox}$, used to track connected components [S1, unofficial].
- `lo`, `hi`: 3D integer arrays defining the bounding box of the region of interest [S1, unofficial].

**Algorithm Steps:**

1.  **Pre-smoothing:** If `preSmooth` is true, apply a Gaussian blur to `img` using `quick_smooth(img, NX, NY, NZ)` [S1, unofficial].

2.  **Intensity Range Calculation:**
    - Initialize `mn = img[0]` and `mx = img[0]`.
    - Iterate through all $n_{vox}$ voxels. For each voxel $i$:
      - `mn = fminf(mn, img[i])`
      - `mx = fmaxf(mx, img[i])`
    - If `mn == mx`, return `EXIT_FAILURE` (constant image intensity causes failure) [S1, unofficial].

3.  **Isolevel Clamping:**
    - If `isolevel <= mn` or `isolevel > mx`, clamp `isolevel` to $0.5 \times (mn + mx)$ and print a warning [S1, unofficial].

4.  **Connected Component Filtering (Optional):**
    - If `onlyLargest` or `fillBubbles` is true:
      - Allocate `mask[nvox]` and initialize to 0.
      - Set `mask[i] = 1` where `img[i] >= isolevel`.
      - Call `bwlabel(mask, 18, dim, onlyLargest, fillBubbles)` to label connected components using 18-connectivity [S1, unofficial].
      - If `fillBubbles` is true:
        - For all $i$ where `mask[i] != 0`, set `img[i] = fmax(img[i], isolevel)` [S1, unofficial].
      - If `onlyLargest` is true:
        - Dilate `mask` by 1 voxel in all directions (`dilate(mask, dim, true)`) to preserve subvoxel edge information [S1, unofficial].
        - For all $i$ where `mask[i] == 0`, set `img[i] = mn` [S1, unofficial].
      - Free `mask` [S1, unofficial].

5.  **Boundary Darkening and Bounding Box Computation:**
    - Initialize `edgeMax = 0.75 \times (mn + isolevel)` [S1, unofficial].
    - Initialize `lo[3] = {NX, NY, NZ}` and `hi[3] = {0, 0, 0}`.
    - Iterate through all voxels $(z, y, x)$:
      - Let $v_x = x + y \times N_X + z \times N_X \times N_Y$.
      - If `img[v_x] >= isolevel`:
        - `lo[0] = MIN(x, lo[0])`
        - `lo[1] = MIN(y, lo[1])`
        - `lo[2] = MIN(z, lo[2])`
        - `hi[0] = MAX(x, hi[0])`
        - `hi[1] = MAX(y, hi[1])`
        - `hi[2] = MAX(z, hi[2])`
      - If $x=0$ or $y=0$ or $z=0$ or $x=N_X-1$ or $y=N_Y-1$ or $z=N_Z-1$:
        - Set `img[v_x] = fminf(edgeMax, img[v_x])` to prevent boundary voxels from being missed by Marching Cubes after cropping [S1, unofficial].

6.  **Bounding Box Expansion:**
    - Expand the bounding box by 1 voxel in all directions to ensure edge cases are captured:
      - For $i \in \{0, 1, 2\}$:
        - `lo[i] = MAX(lo[i] - 1, 0)`
        - `hi[i] = MIN(hi[i] + 2, dim[i])` [S1, unofficial].

7.  **Marching Cubes Extraction:**
    - Run `marchingCubes(img, dim, lo, hi, originalMC, isolevel, &pts, &tris, &npt, &ntri)` on the cropped volume defined by `lo` and `hi` [S1, unofficial].
    - Return `EXIT_FAILURE` on error [S1, unofficial].

8.  **Post-processing:**
    - Unify vertices: `npt = unify_vertices(&pts, tris, npt, ntri, verbose)` [S1, unofficial].
    - Remove degenerate triangles: `ntri = remove_degenerate_triangles(pts, &tris, ntri, verbose)` [S1, unofficial].

9.  **Output Assignment:**
    - Assign `*t = tris`, `*p = pts`, `*nt = ntri`, `*np = npt` [S1, unofficial].
    - Return `EXIT_SUCCESS` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def preprocessed_marching_cubes(img, dim, isolevel, preSmooth=False, onlyLargest=False, fillBubbles=False):
    NX, NY, NZ = dim
    nvox = NX * NY * NZ
    
    # Step 1: Pre-smoothing
    if preSmooth:
        img = quick_smooth(img, NX, NY, NZ)
        
    # Step 2: Intensity Range
    mn = img[0]
    mx = img[0]
    for i in range(nvox):
        mn = min(mn, img[i])
        mx = max(mx, img[i])
        
    if mn == mx:
        return EXIT_FAILURE
        
    # Step 3: Isolevel Clamping
    if isolevel <= mn or isolevel > mx:
        isolevel = 0.5 * (mn + mx)
        print("Warning: Isolevel clamped to average intensity")
        
    # Step 4: Connected Component Filtering
    if onlyLargest or fillBubbles:
        mask = [0] * nvox
        for i in range(nvox):
            if img[i] >= isolevel:
                mask[i] = 1
                
        # bwlabel implementation details are assumed to be available
        # 18-connectivity for 3D
        bwlabel(mask, 18, dim, onlyLargest, fillBubbles)
        
        if fillBubbles:
            for i in range(nvox):
                if mask[i] != 0:
                    img[i] = max(img[i], isolevel)
                    
        if onlyLargest:
            # Dilate mask by 1 voxel
            dilated_mask = dilate(mask, dim, True)
            for i in range(nvox):
                if dilated_mask[i] == 0:
                    img[i] = mn
        # Free mask memory if managed manually
        del mask
        
    # Step 5: Boundary Darkening and Bounding Box
    edgeMax = 0.75 * (mn + isolevel)
    lo = [NX, NY, NZ]
    hi = [0, 0, 0]
    
    for z in range(NZ):
        for y in range(NY):
            for x in range(NX):
                vx = x + y * NX + z * NX * NY
                if img[vx] >= isolevel:
                    lo[0] = min(x, lo[0])
                    lo[1] = min(y, lo[1])
                    lo[2] = min(z, lo[2])
                    hi[0] = max(x, hi[0])
                    hi[1] = max(y, hi[1])
                    hi[2] = max(z, hi[2])
                    
                # Check boundary
                if x == 0 or y == 0 or z == 0 or x == NX - 1 or y == NY - 1 or z == NZ - 1:
                    img[vx] = min(edgeMax, img[vx])
                    
    # Step 6: Expand Bounding Box
    for i in range(3):
        lo[i] = max(lo[i] - 1, 0)
        hi[i] = min(hi[i] + 2, dim[i])
        
    # Step 7: Marching Cubes
    # Note: marchingCubes expects a cropped view or handles the bounds internally
    # Assuming it takes the full img but only processes cells within lo/hi
    pts, tris, npt, ntri = marchingCubes(img, dim, lo, hi, originalMC, isolevel)
    
    if npt is None:
        return EXIT_FAILURE
        
    # Step 8: Post-processing
    npt = unify_vertices(pts, tris, npt, ntri)
    ntri = remove_degenerate_triangles(pts, tris, ntri)
    
    # Step 9: Output
    return tris, pts, ntri, npt, EXIT_SUCCESS
```

**Parameters:**
- `preSmooth`: Boolean flag to apply Gaussian blur. Typical value: `false` [S1, unofficial].
- `onlyLargest`: Boolean flag to extract only the largest connected component. Typical value: `false` [S1, unofficial].
- `fillBubbles`: Boolean flag to fill internal voids within the largest component. Typical value: `false` [S1, unofficial].
- `isolevel`: Scalar threshold for surface extraction. Default: $0.5 \times (mn + mx)$ if out of range [S1, unofficial].
- `edgeMax`: Intensity cap for boundary voxels, calculated as $0.75 \times (mn + isolevel)$ [S1, unofficial].

**Pitfalls:**
- Constant image intensity (`mn == mx`) causes Marching Cubes to fail; handled by returning `EXIT_FAILURE` [S1, unofficial].
- Out-of-range `isolevel` causes incorrect surface extraction; handled by clamping to $0.5 \times (mn + mx)$ [S1, unofficial].
- Boundary voxels may be missed by Marching Cubes if not darkened; handled by setting boundary voxels to `fminf(edgeMax, img[vx])` [S1, unofficial].
- Subvoxel edge preservation is lost when cropping; handled by dilating the mask by one voxel before extracting the largest component [S1, unofficial].

## Subtopics

### [Unbounded TSDF Mesh Extraction](6832-unbounded-tsdf-mesh-extraction.md)
Extracts a triangle mesh from an unbounded scene represented by 3D Gaussians by contracting the infinite space into a bounded unit volume, performing adaptive Truncated Signed Distance Function (TSDF) fusion, and applying Marching Cubes. It sets itself apart from standard bounded extraction by handling scenes with no predefined depth limits through space contraction and adaptive truncation, avoiding the need for a single massive global grid.

### [Connected Component Filtering](10789-connected-component-filtering.md)
Connected Component Filtering segments a triangle mesh to retain only the largest connected component of faces, discarding isolated geometry artifacts such as floaters and disconnected parts. It builds on [Preprocessed Marching Cubes](3765-preprocessed-marching-cubes.md) by isolating the primary object and filling internal bubbles, distinguishing itself from [Unbounded TSDF Mesh Extraction](6832-unbounded-tsdf-mesh-extraction.md) and [Marching Cubes with Histogram Pyramids](12322-marching-cubes-with-histogram-pyramids.md) through explicit topological clustering of the resulting polydata.

### [Marching Cubes with Histogram Pyramids](12322-marching-cubes-with-histogram-pyramids.md)
A 3D surface reconstruction method that uses histogram pyramids to reduce redundant voxels (unused for generating the 3D surface) before applying the marching cubes algorithm, resulting in smoother surfaces and lower mesh density. It differs from standard Marching Cubes by pre-filtering the voxel grid to eliminate sparse or irrelevant data, thereby reducing the computational load and surface roughness associated with high-density meshes.

## Operating characteristics
- **Time Complexity:** Dominated by the connected component labeling (`bwlabel`) if enabled, which is $O(N_{vox})$, and the Marching Cubes extraction, which is $O(N_{cells})$ where $N_{cells} \approx N_X N_Y N_Z$. The boundary darkening and bounding box computation are also linear in the number of voxels.
- **Memory Complexity:** Requires $O(N_{vox})$ for the input volume, the mask array (if filtering is enabled), and the output mesh. The mask array is temporary and can be freed after filtering.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Performance scales linearly with the number of voxels. The bounding box optimization reduces the number of cells processed by Marching Cubes if the object is small relative to the volume.
- **Strengths:** Robust handling of noisy data via smoothing, ability to isolate primary objects and fill voids, and prevention of boundary artifacts.
- **Limitations:** Requires additional preprocessing steps which increase complexity. The choice of connectivity (18-connectivity) and dilation parameters can affect the final topology.
- **When to use:** When the input volume contains noise, multiple disconnected components, or internal voids that need to be filled, and when boundary artifacts are a concern.
- **When not to use:** When real-time performance is critical and preprocessing overhead is unacceptable, or when the input volume is already clean and well-segmented.

## Background
Preprocessed Marching Cubes is an implementation variant of the Marching Cubes algorithm that incorporates preprocessing steps to improve the quality and robustness of the extracted mesh. It is commonly used in medical imaging and 3D reconstruction pipelines where input data may be noisy or contain artifacts. The method is associated with the `niimath` tool and the `meshify` utility [S1, unofficial].

## Sources
[S1] TRTUZ8/TRTUZ8.github.io, https://github.com/TRTUZ8/TRTUZ8.github.io, fetched 2026-10-28 (unofficial)
