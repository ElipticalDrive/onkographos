# Size Volume Computation

Also called: Correa and Ma Size Volume, Size Attribute, Tamanho

## Overview
Size Volume Computation generates a tertiary volumetric attribute representing the local scale or "size" of features within a dataset, enabling size-based transfer functions in Direct Volume Rendering (DVR) [S1, S2, unofficial]. It addresses the difficulty of defining mathematical descriptors for size variations, which are visually intuitive but mathematically complex [S1, unofficial]. Unlike sibling methods that compute density or coverage, this method identifies blob-like structures across multiple scales to assign a size value to each voxel, distinguishing it from [Interior Voxelize](995-interior-voxelize.md) and [Shell-based Voxelization](1748-shell-based-voxelization.md) by focusing on multiscale feature characterization rather than geometric containment or boundary representation.

## How it works
**Inputs and Outputs:**
- Input: A scalar volume $V$ with dimensions $(N_x, N_y, N_z)$. Optionally, an intensity range $[I_{min}, I_{max}]$ to restrict scale-space computation to specific features [S2, unofficial].
- Output: A size volume $S$ with the same dimensions, where each voxel contains the size value of the largest detected feature at that location.

**Data Structures:**
- Input Volume $V$: 3D array of floats.
- Size Volume $S$: 3D array of floats, initialized to zero or background value.
- Blob List: A list of tuples $(x, y, z, t)$, where $(x, y, z)$ are voxel coordinates and $t$ is the detected scale (size) value.

**Steps:**
1. **Scale-Space Computation**: Construct a scale space from the input volume using anisotropic diffusion. This step enhances localization of features compared to standard Gaussian smoothing [S2, unofficial]. Let $V_0 = V$. For each scale level $i$, compute $V_i$ by applying anisotropic diffusion to $V_{i-1}$ with a scale parameter $\sigma_i$. The scale parameter $t$ is related to $\sigma$ by $t = \sigma^2/2$ [S2, G, unofficial]. The scale space is a 4D structure $V(x, y, z, t)$.

2. **Scale Detection**: Detect blobs as local maxima in both spatial and scale domains using the classical normalized Laplacian kernel. For each voxel $(x, y, z)$ and scale $t$, compute the Laplacian $\nabla^2 V(x, y, z, t)$. Normalize by the scale to ensure scale invariance: $L_{norm}(x, y, z, t) = t \cdot \nabla^2 V(x, y, z, t)$ [S2, unofficial]. Identify local maxima in the 4D space. Store detected blobs as tuples $(x, y, z, t)$ in the Blob List.

3. **Back Projection**: For each voxel $(x, y, z)$ in the output volume $S$, determine the size value by back-projecting the detected blobs. Use Shepard's interpolation with Wendland polynomials to assign the size value $t$ from the nearest detected blob $(x_b, y_b, z_b, t_b)$ to the voxel [S2, unofficial]. The interpolation weight $w_i$ for a blob $i$ at distance $d_i$ is computed using a Wendland polynomial $w(d_i)$, and the interpolated size is $S(x, y, z) = \frac{\sum w_i t_i}{\sum w_i}$. If no blob is detected near a voxel, $S(x, y, z)$ remains at its initial value.

4. **Masking**: For each voxel, if multiple blobs are detected at different scales, keep only the largest size value $t_{max}$ and mask out smaller features [S2, unofficial]. This ensures that each voxel represents the dominant feature size.

**Parameters:**
- Scale levels: Number of scale steps in the anisotropic diffusion process. Not given in the sources.
- Intensity range: Optional $[I_{min}, I_{max}]$ to restrict scale-space computation to specific intensity values, preventing smaller features from being masked by larger ones in different intensity ranges [S2, unofficial].

**Edge Cases and Pitfalls:**
- Smaller features get masked out by larger ones if they share the same spatial location but different scales [S2, unofficial]. Mitigation: Use intensity-range-specific scale spaces [S2, unofficial].
- Precision issues in anisotropic diffusion may affect localization accuracy [S2, G, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_size_volume(V, intensity_range=None):
    # Step 1: Scale-Space Computation
    # Apply anisotropic diffusion to create scale space
    # V_scale is a 4D array [x, y, z, t]
    V_scale = anisotropic_diffusion(V)
    
    # Step 2: Scale Detection
    # Detect blobs as local maxima in spatial and scale domains
    blob_list = []
    for x, y, z in voxels(V):
        for t in scales(V_scale):
            laplacian = normalized_laplacian(V_scale, x, y, z, t)
            if is_local_maxima(V_scale, x, y, z, t, laplacian):
                blob_list.append((x, y, z, t))
    
    # Step 3: Back Projection
    S = zeros_like(V)
    for x, y, z in voxels(V):
        # Find nearest blob using Shepard's interpolation with Wendland polynomials
        weights = []
        values = []
        for bx, by, bz, bt in blob_list:
            d = distance((x, y, z), (bx, by, bz))
            w = wendland_polynomial(d)
            weights.append(w)
            values.append(bt)
        
        if weights:
            S[x, y, z] = sum(w * v for w, v in zip(weights, values)) / sum(weights)
    
    # Step 4: Masking
    # Keep only the largest size value at each voxel
    # This is implicitly handled by back projection if blobs are sorted by size
    # or by post-processing to take max over overlapping scales
    return S

def anisotropic_diffusion(V):
    # Apply anisotropic diffusion to create scale space
    # Not given in the sources. Placeholder for implementation.
    pass

def normalized_laplacian(V_scale, x, y, z, t):
    # Compute normalized Laplacian
    # Not given in the sources. Placeholder for implementation.
    pass

def is_local_maxima(V_scale, x, y, z, t, laplacian):
    # Check if laplacian is local maximum in 4D space
    # Not given in the sources. Placeholder for implementation.
    pass

def wendland_polynomial(d):
    # Compute Wendland polynomial weight
    # Not given in the sources. Placeholder for implementation.
    pass

def distance(p1, p2):
    # Euclidean distance
    return sqrt(sum((a - b) ** 2 for a, b in zip(p1, p2)))
```

## Subtopics

### Intensity-Range-Specific Scale Space
A variant where scale-space computation is restricted to a user-defined intensity range, preventing smaller features from being masked by larger ones in different intensity ranges. Choose this when features of interest have distinct intensity profiles. Link to [Size Volume Computation](52-size-volume-computation.md).

## Operating characteristics
- **Time Complexity**: Depends on the number of scale levels and voxels. Anisotropic diffusion is iterative and may be slow. Scale detection involves 4D local maxima search, which is computationally intensive. Not given in the sources.
- **Memory Complexity**: Requires storing the 4D scale space and blob list. Not given in the sources.
- **Strengths**: Provides intuitive size-based transfer functions for feature categorization. Effective for blob-like structures.
- **Limitations**: Smaller features may be masked by larger ones. Computationally expensive due to 4D processing.
- **When to Use**: When size-based feature distinction is critical, such as in medical imaging for tumor vs. vessel differentiation.
- **When Not to Use**: For real-time applications or when features are not blob-like.

## Background
Originated from Correa and Ma (2008), who adapted size concepts from computer vision to DVR context [S1, unofficial]. The method analyzes multiscale fields and pyramid schemes to categorize objects in a volume [S1, unofficial]. It is used in medical imaging, such as brain MRI for aneurysm detection, where size-based transfer functions help distinguish tumors from blood vessels [S1, unofficial].

## Sources
[S1] A volume exploration approach based on clustering and dimensional reduction to support the definition of multidimensional transfer functions, http://hdl.handle.net/11449/153618, fetched 2026-10-26 (unofficial)
[S2] MULTIVARIATE TRANSFER FUNCTION DESIGN, https://www.sci.utah.edu/~lzhou/dissertation_final_zhoul.pdf, fetched 2026-10-27 (unofficial)
