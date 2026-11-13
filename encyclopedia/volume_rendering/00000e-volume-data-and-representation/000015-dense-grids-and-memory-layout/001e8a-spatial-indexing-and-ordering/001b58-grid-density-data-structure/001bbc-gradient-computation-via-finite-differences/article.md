# Gradient Computation via Finite Differences

Also called: Central Difference Gradient, Central Differences, Finite Difference Gradient, Volume Gradient Estimation (Central Differences)

## Overview
Gradient Computation via Finite Differences estimates the surface normal of a scalar volume at sample positions by approximating the spatial derivative of the density field using neighboring voxel values [S1, S3, unofficial][S4, book]. This method solves the problem of providing directional lighting cues (such as Phong shading or global illumination) and enabling multidimensional transfer functions, which require local surface orientation information [S1, unofficial][S4, book]. It sets itself apart from [Analytic Gradient Computation](2167-analytic-gradient-computation.md) by relying on discrete sampling rather than closed-form derivatives of basis functions, and from [Gradient Estimation from Samples](2180-gradient-estimation-from-samples.md) by using a fixed, regular grid structure with a minimal stencil [S3, unofficial][S22].

## How it works
**Inputs:** A scalar volume $V$ defined on a 3D grid with dimensions $W \times H \times D$ (width, height, depth). The volume data is typically stored in row-major order (x fastest, y middle, z slowest) [S10, S12, unofficial]. Physical spacing between voxels along axes $x, y, z$ is denoted by $h_x, h_y, h_z$ [S4, S28, book].

**Outputs:** A gradient vector field $\nabla V$, where each voxel $(i, j, k)$ has an associated 3D vector $(g_x, g_y, g_z)$ [S8, S15, unofficial].

**Data Structures:**
*   **Input Volume:** 1D array or 3D texture of size $W \times H \times D$.
*   **Gradient Buffer:** 1D array of size $W \times H \times D \times 3$ (interleaved RGB) or a 3D texture (e.g., `RGBA8_SNORM` or `RG8`) [S8, S17, unofficial]. If using a texture, signed normals in $[-1, 1]$ are often encoded to $[0, 1]$ via $enc = n \cdot 0.5 + 0.5$ and quantized to 8-bit integers [S17, unofficial].

**Formulas:**
The central difference approximation for the gradient component along axis $a$ at voxel $(i, j, k)$ is:
$$ g_a = \frac{V_{next} - V_{prev}}{2 h_a} $$
where $V_{next}$ and $V_{prev}$ are the scalar values at the immediate neighbors along axis $a$, and $h_a$ is the physical distance between voxels along that axis [S4, book][S27, unofficial][S28, book].

Explicitly:
$$ g_x = \frac{V(i+1, j, k) - V(i-1, j, k)}{2 h_x} $$
$$ g_y = \frac{V(i, j+1, k) - V(i, j-1, k)}{2 h_y} $$
$$ g_z = \frac{V(i, j, k+1) - V(i, j, k-1)}{2 h_z} $$

The resulting gradient vector is normalized to obtain the surface normal $\vec{N}$:
$$ \vec{N} = \frac{(g_x, g_y, g_z)}{\sqrt{g_x^2 + g_y^2 + g_z^2}} $$
If the magnitude is zero or below a threshold (e.g., $10^{-6}$), a fallback normal (e.g., $(0, 1, 0)$) or zero vector is returned [S7, S17, unofficial].

**Parameters:**
*   **Step Size ($h$):** The physical distance between voxels. If the grid is unit-spaced, $h=1$ [S4, S28, book].
*   **Offset ($n$):** Some implementations use a larger offset $n$ for the difference (e.g., $V(i+n) - V(i-n)$), dividing by $2n$ [S9, S14, unofficial].
*   **Smoothing:** A box filter of size $N \times N \times N$ may be applied to the computed gradients to reduce noise [S9, unofficial].

**Steps:**
1.  Iterate through every voxel $(i, j, k)$ in the volume.
2.  For each axis ($x, y, z$), identify the neighbor indices.
3.  Handle boundary conditions:
    *   If the neighbor is within bounds, use the central difference formula.
    *   If the neighbor is out of bounds (at the volume edge), use a one-sided difference (forward or backward) or clamp the coordinate to the boundary value [S5, S26, S27, unofficial].
4.  Compute the raw gradient components $(g_x, g_y, g_z)$.
5.  Normalize the vector.
6.  Store the result in the gradient buffer.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_gradient(volume, width, height, depth, hx, hy, hz):
    """
    Computes the gradient vector field for a scalar volume using central differences.
    volume: 1D array of size width*height*depth, row-major (x fastest).
    hx, hy, hz: physical spacing along each axis.
    Returns: 1D array of size width*height*depth*3, interleaved (gx, gy, gz).
    """
    num_voxels = width * height * depth
    gradients = [0.0] * (num_voxels * 3)
    
    # Precompute reciprocal of 2*h for efficiency
    inv_2hx = 1.0 / (2.0 * hx)
    inv_2hy = 1.0 / (2.0 * hy)
    inv_2hz = 1.0 / (2.0 * hz)
    
    for k in range(depth):
        for j in range(height):
            for i in range(width):
                # Linear index for current voxel
                idx = i + j * width + k * width * height
                
                # --- X Component ---
                # Determine neighbors
                if i > 0 and i < width - 1:
                    # Central difference
                    prev_val = volume[idx - 1]
                    next_val = volume[idx + 1]
                    gx = (next_val - prev_val) * inv_2hx
                elif i == 0:
                    # Forward difference (boundary)
                    next_val = volume[idx + 1]
                    curr_val = volume[idx]
                    gx = (next_val - curr_val) / hx
                else: # i == width - 1
                    # Backward difference (boundary)
                    curr_val = volume[idx]
                    prev_val = volume[idx - 1]
                    gx = (curr_val - prev_val) / hx
                
                # --- Y Component ---
                if j > 0 and j < height - 1:
                    # Central difference
                    # Index calculation: y changes every 'width' elements
                    prev_val = volume[idx - width]
                    next_val = volume[idx + width]
                    gy = (next_val - prev_val) * inv_2hy
                elif j == 0:
                    next_val = volume[idx + width]
                    curr_val = volume[idx]
                    gy = (next_val - curr_val) / hy
                else: # j == height - 1
                    curr_val = volume[idx]
                    prev_val = volume[idx - width]
                    gy = (curr_val - prev_val) / hy
                
                # --- Z Component ---
                if k > 0 and k < depth - 1:
                    # Central difference
                    # Index calculation: z changes every 'width*height' elements
                    prev_val = volume[idx - width * height]
                    next_val = volume[idx + width * height]
                    gz = (next_val - prev_val) * inv_2hz
                elif k == 0:
                    next_val = volume[idx + width * height]
                    curr_val = volume[idx]
                    gz = (next_val - curr_val) / hz
                else: # k == depth - 1
                    curr_val = volume[idx]
                    prev_val = volume[idx - width * height]
                    gz = (curr_val - prev_val) / hz
                
                # --- Normalization ---
                length_sq = gx*gx + gy*gy + gz*gz
                if length_sq > 1e-12:
                    length = length_sq ** 0.5
                    gx /= length
                    gy /= length
                    gz /= length
                else:
                    gx = 0.0
                    gy = 0.0
                    gz = 0.0 # Or fallback normal [S7, unofficial]
                
                # Store interleaved
                gradients[idx * 3] = gx
                gradients[idx * 3 + 1] = gy
                gradients[idx * 3 + 2] = gz
                
    return gradients
```

**Edge Cases and Pitfalls:**
*   **Boundaries:** The gradient is undefined at the volume boundary. Common solutions include clamping coordinates (repeating edge values), using one-sided differences, or padding the volume [S4, book][S5, S26, unofficial]. Failure to handle boundaries causes out-of-bounds memory access [S10, S12, unofficial].
*   **Zero Magnitude:** In flat regions or constant values, the gradient magnitude is zero. Normalization fails; a default normal or zero vector must be returned [S7, S17, unofficial].
*   **Noise:** Central differences are sensitive to high-frequency noise in the data, potentially causing visual artifacts [S29, unofficial]. Smoothing filters may be required [S9, unofficial].
*   **Precision:** When using fixed-point textures (e.g., 8-bit), quantization errors can distort normals. Signed normalized formats (`SNORM`) are preferred [S17, unofficial].

## Subtopics

### [Gradient Map Generation](1204-gradient-map-generation.md)
Gradient Map Generation is a GPU-accelerated compute shader pass that calculates the gradient vector for every voxel in a scalar volume and stores the results in a dedicated 3D image buffer. It solves the problem of efficiently preparing gradient data for subsequent volume rendering techniques, such as lighting and shading, by decoupling gradient computation from the final ray-marching or projection pass. Unlike [Analytic Gradient Computation](2167-analytic-gradient-computation.md) which derives gradients from mathematical functions, or [Precomputed Gradient Volume](2425-precomputed-gradient-volume.md) which may use offline tools, this method performs the calculation dynamically on the GPU using finite differences. It differs from [Gradient-based Diffuse Lighting with Alpha Blending](1257-gradient-based-diffuse-lighting-with-alpha-blending.md) by focusing solely on the storage of gradient data rather than applying lighting models immediately.

### [Gradient-based Diffuse Lighting with Alpha Blending](1257-gradient-based-diffuse-lighting-with-alpha-blending.md)
This method computes diffuse lighting for a scalar volume by estimating surface normals from voxel gradients and compositing the lit color into a framebuffer using front-to-back alpha blending. It integrates gradient estimation, lighting application, and compositing into a single per-voxel step rather than storing precomputed gradients separately.

### [Analytic Gradient Computation](2167-analytic-gradient-computation.md)
Analytic Gradient Computation computes the 3D gradient vector for local shading (BRDF) in volume rendering by evaluating the exact first-order partial derivatives of the basis reconstruction function. It sets itself apart from [Gradient Computation via Finite Differences](1077-gradient-computation-via-finite-differences.md) by requiring no additional memory accesses or ray traversals, relying solely on the data values already loaded for the scalar evaluation. Unlike [Gradient Map Generation](1204-gradient-map-generation.md) which precomputes gradients, this method computes them on-the-fly during ray marching.

### [Gradient Estimation from Samples](2180-gradient-estimation-from-samples.md)
Gradient Estimation from Samples computes surface normals directly from scalar field samples taken during volume ray casting, enabling Phong-based shading without precomputing a separate gradient volume. It differs from [Precomputed Gradient Volume](2425-precomputed-gradient-volume.md) by avoiding the memory overhead and preprocessing time of storing explicit gradient vectors, and from [Analytic Gradient Computation](2167-analytic-gradient-computation.md) by relying on discrete finite differences rather than closed-form derivatives. This method is primarily used in interactive GPU-based volume rendering to enhance shape perception through diffuse and specular lighting cues.

### [Precomputed Gradient Volume](2425-precomputed-gradient-volume.md)
Precomputed Gradient Volume is a volume rendering acceleration technique that trades memory for rendering speed by calculating and storing the 3D gradient vector for every voxel in a scalar volume during a preprocessing step. This allows the rendering engine to sample surface normals directly from memory during raycasting, avoiding the computational cost of finite difference calculations per sample. It distinguishes itself from on-the-fly gradient estimation by requiring significant additional storage (e.g., more than 200 megabytes for a $256 \times 256 \times 256$ dataset).

### [Gradient-based Surface Projection](2668-gradient-based-surface-projection.md)
Gradient-based surface projection visualizes interfaces between regions of different density by projecting through a matrix of gradient magnitude values derived from voxel density. It differs from standard volume rendering by treating the volume as a collection of internal surfaces defined by normals and strengths, rather than a continuous medium with absorption and emission. This method allows for the visualization of diffuse transitions and sub-voxel surface accuracy that binary representations cannot achieve.

### [Per-Region Gradient Averaging](2715-per-region-gradient-averaging.md)
Per-Region Gradient Averaging computes the gradient magnitude for each voxel in a scalar volume by aggregating directional gradients across multiple defined scalar regions. It distinguishes itself from standard finite difference methods by using a binary neighbor check to detect region boundaries, assigning high weights to neighbors within the same region to enforce spatial coherence. This method is primarily used to smooth learned volume representations and prevent high-frequency noise, serving as a preprocessing step for regularization techniques like Total Variation Gradient Addition[2877-total-variation-gradient-addition.md].

### [Gradient Magnitude Computation](3184-gradient-magnitude-computation.md)
Gradient Magnitude Computation derives a scalar volumetric attribute that measures the local rate of intensity variation by associating a value with the neighborhood of volumetric cells. It serves as a critical input for Transfer Functions (TFs), enabling the identification of edges and surfaces in addition to monotonic regions. Unlike the parent method which outputs a vector field, this method outputs a scalar field representing the Euclidean norm of the gradient, allowing TFs to distinguish high-contrast boundaries from smooth interior regions.

### [Scanline Gradient Computation](3582-scanline-gradient-computation.md)
Scanline Gradient Computation is a memory-efficient method for computing 3D gradient vectors and surface normals for scalar volume data by processing one scanline at a time. It solves the problem of high memory bandwidth and storage requirements when computing derivatives for large volumes by maintaining only a sliding window of three slices in memory, making it suitable for volumes that exceed available RAM. It sets itself apart from batch processing methods like [VolPack Volume Normals Computation](3126-volpack-volume-normals-computation.md) by streaming data slice-by-slice rather than requiring the entire scalar array in memory.

### [Finite Difference Stencils (Central/Forward/Backward)](3858-finite-difference-stencils-central-forward-backward.md)
Finite difference stencils approximate the spatial gradient of a scalar volume field by computing weighted differences between voxel values at discrete offsets. This method solves the problem of estimating surface normals for lighting and opacity calculations in volume rendering when analytical derivatives are unavailable. It is distinguished from [Analytic Gradient Computation](2167-analytic-gradient-computation.md) by its reliance on discrete sampling rather than closed-form expressions, and from [Gradient Estimation from Samples](2180-gradient-estimation-from-samples.md) by its fixed, regular grid structure.

### [Linearly Separable Piecewise Polynomial Interpolation Gradient](5841-linearly-separable-piecewise-polynomial-interpolation-gradie.md)
This method estimates the 3D gradient vector at each voxel in a scalar volume by applying optimized linearly separable filters along each axis, rather than computing gradients via standard finite differences and then interpolating. It is distinguished from sibling methods by performing only one interpolation step per gradient component, which yields higher quality results for illumination calculations where gradients are normalized to compute surface normals. Unlike [Analytic Gradient Computation](2167-analytic-gradient-computation.md) which requires analytic derivatives of the transfer function, or [Gradient Estimation from Samples](2180-gradient-estimation-from-samples.md) which uses regression over a local neighborhood, this approach relies on fixed coefficients derived from piecewise polynomial interpolation theory.

### [Second Derivative Computation](6353-second-derivative-computation.md)
Second Derivative Computation computes second-order spatial derivatives (the Hessian matrix or its scalar contractions) of a scalar field defined on a grid. It solves the problem of quantifying local curvature, boundary sharpness, and structural homogeneity, which are essential for regularization, surface extraction, and physical simulation. Unlike [Gradient Computation via Finite Differences](1077-gradient-computation-via-finite-differences.md), which yields the first-order vector field, this family computes the rate of change of that gradient. It differs from [Analytic Gradient Computation](2167-analytic-gradient-computation.md) by relying on discrete stencil approximations rather than closed-form derivatives of basis functions, and from [Gradient Estimation from Samples](2180-gradient-estimation-from-samples.md) by operating on structured grid data rather than unstructured point clouds.

### Gradient-Based Flow Direction Calculation
Deterministic calculation of normalized chunk-local X/Z flow vectors for fluid surfaces based on the gradient of water levels of the same fluid type [S32, unofficial]. It calculates flow direction from the gradient of same-type fluid surface heights. If a same-type fluid exists one level below in an adjacent empty cell, it is treated as flow over a step. Still water has a flow direction of `[0, 0]`. Unloaded adjacent chunks are treated as static (no flow) rather than fabricating flow [S32, unofficial].

### Spatial SDF Gradient Computation
Computation of the spatial gradient of the SDF at each site by assuming a constant gradient within each tetrahedron and averaging the gradients of incident tetrahedra weighted by volume [S33]. It estimates the SDF at site $i$ within tetrahedron $j$ using a first-order approximation based on the tetrahedron's centroid, local gradient, and site position. It obtains the local gradient $\nabla \phi^{(j)}$ for each tetrahedron by minimizing the mean squared difference between the actual SDF values at the sites and the estimated values, solved efficiently by a linear system. The final spatial gradient at site $s_i$ is computed by averaging the local gradients $\nabla \phi^{(j)}$ of all incident tetrahedra $j$, weighted by their volumes $V_j$ [S33].

### Hardware Gradient Calculator Pipeline
A hardware architecture designed to calculate gradient values at voxel data rate for shading. It uses parallel pipelines for X, Y, and Z components. The pipeline for Y gradient calculation has three stages: 1) Column total calculation using a Triple Input Adder (TIA) into a column total register. 2) Side total calculation using a second TIA into a side total register. 3) Y gradient component calculation by finding the difference between side totals from the Y and X gradient calculators. The TIA implementation uses an 8-bit ripple carry adder with a minimum clock cycle of 8ns, which is the slowest stage. A single processing element in non-aggressive technology may not achieve video rate; parallel processing elements dividing volume data into equal volumes are required for approximately linear speedup [S34].

### Perceptual Gradient Computation
A method to compute gradients and opacities for multi-channel data (e.g., RGB) by converting to a perceptionally uniform color space to overcome RGB perceptional nonlinearities [S35, unofficial]. It converts RGB data into the L*u*v* space, calculates the gradient via central differences replacing voxel densities with calculated color distances, and computes opacities using functions of higher-level gradients to highlight textural and structural aspects. The direction of the normal vector cannot be determined with this method, though this is not considered a limiting factor in practice [S35, unofficial].

### Density Gradient
A technique that calculates the vertex gradient based on opacity rather than scalar values to prevent normal flipping errors in shading [S36]. Density gradient magnitude is easier to normalize because opacity has a fixed range, whereas scalar value ranges vary. Scalar gradients may incidentally flip the surface normal, causing shadows to be cast at the wrong location if only one side of the surface has reflection. Enabling both gradient opacity and density gradient leads to three times more gradient lookups. If gradients are computed at runtime, there is a linear increase in computational time. Gradient opacity disables rendering of voxels when their gradient magnitude is below a threshold, accentuating edges and surfaces without precise opacity transfer function manipulation [S36].

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of voxels, as each voxel requires a constant number of reads and arithmetic operations [S8, unofficial].
*   **Memory Complexity:** $O(N)$ for storing the gradient field if precomputed, or $O(1)$ if computed on-the-fly during ray marching [S8, unofficial].
*   **Performance:** The central difference operator requires only 6 sampling points per voxel, making it computationally inexpensive compared to higher-order operators like Sobel3D or Zucker-Hummel, which use more voxels and offer smoother but more expensive gradients [S3, unofficial][S22].
*   **Quality:** Provides a worse description of local surface structure compared to operators using more voxels (e.g., Sobel3D) [S3, unofficial]. It displays substantial voxel artifacts from coarse data resolution compared to regression methods [S22]. Visual artifacts are similar to those resulting from resampling with trilinear interpolation [S4, book][S30].
*   **Use Cases:** Ideal for real-time volume rendering where performance is critical and high-frequency noise is not a primary concern. Suitable for on-the-fly shading in ray casters [S3, unofficial][S22].
*   **Limitations:** Sensitive to noise; boundary handling is non-trivial and can introduce artifacts if not carefully implemented [S4, book][S5, unofficial].

## Background
The method originates from standard numerical analysis techniques for approximating derivatives. In volume rendering, it was popularized by Levoy for computing surface normals for Phong shading and depth cueing in direct volume rendering [S2, unofficial]. It is widely used in conjunction with transfer functions and local illumination models to enhance the perception of shape and texture in scalar volumes [S1, unofficial][S4, book]. It is implemented in various graphics libraries and tools, such as CGAL for isosurfacing [S31] and NVIDIA's GPU Gems [S4, book].

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] Direct Volume Rendering (DVR): Ray-casting | PPT - SlideShare, https://www.slideshare.net/slideshow/direct-volume-rendering-dvr-raycasting/265104120, fetched 2026-10-26 (unofficial)
[S3] Volume Raycasting mit OpenCL, https://arxiv.org/abs/1609.01317v1, fetched 2026-10-26 (unofficial)
[S4] Chapter 39. Volume Rendering Techniques | NVIDIA Developer, https://developer.nvidia.com/gpugems/gpugems/part-vi-beyond-triangles/chapter-39-volume-rendering-techniques, fetched 2026-10-26 (book)
[S5] joshcodd/volume-rendering, https://github.com/joshcodd/volume-rendering, fetched 2026-10-26 (unofficial)
[S6] CalamityMichL/DVR_ESS_PDM, https://github.com/CalamityMichL/DVR_ESS_PDM, fetched 2026-10-26 (unofficial)
[S7] xwghua/clearvolume-plus, https://github.com/xwghua/clearvolume-plus, fetched 2026-10-26 (unofficial)
[S8] extrovert7986/Scientific-visualization-Ray-casting-volume-rendering, https://github.com/extrovert7986/Scientific-visualization-Ray-casting-volume-rendering, fetched 2026-10-26 (unofficial)
[S9] CSE-333-Computer-Graphics-2020/VolumeRayCasting, https://github.com/CSE-333-Computer-Graphics-2020/VolumeRayCasting, fetched 2026-10-26 (unofficial)
[S10] thoniorf/volumeraycasting, https://github.com/thoniorf/volumeraycasting, fetched 2026-10-26 (unofficial)
[S11] Real-Time Mono- and Multi-Volume Rendering of Large Medical Datasets on Standard PC Hardware, https://doi.org/10.2312/diss.20058170, fetched 2026-10-26 (unofficial)
[S12] Scrawk/Marching-Cubes-On-The-GPU, https://github.com/Scrawk/Marching-Cubes-On-The-GPU, fetched 2026-10-26 (unofficial)
[S13] Magnus2/MeshReconstruction, https://github.com/Magnus2/MeshReconstruction, fetched 2026-10-26 (unofficial)
[S14] jose-villegas/IllustrativeVolumeRendering, https://github.com/jose-villegas/IllustrativeVolumeRendering, fetched 2026-10-26 (unofficial)
[S15] tobiasosborne/Lyr.jl, https://github.com/tobiasosborne/Lyr.jl, fetched 2026-10-26 (unofficial)
[S16] fargiolas/qvrc, https://github.com/fargiolas/qvrc, fetched 2026-10-26 (unofficial)
[S17] sho3la/GPUGemsVolumeRendering, https://github.com/sho3la/GPUGemsVolumeRendering, fetched 2026-10-26 (unofficial)
[S18] faviasono/raycastingrendering, https://github.com/faviasono/raycastingrendering, fetched 2026-10-26 (unofficial)
[S19] KaanAres10/Volume-Rendering, https://github.com/KaanAres10/Volume-Rendering, fetched 2026-10-26 (unofficial)
[S20] thebusytypist/cudc, https://github.com/thebusytypist/cudc, fetched 2026-10-26 (unofficial)
[S21] [PDF] The VolumePro Real-Time Ray-Casting System, https://www.merl.com/publications/docs/TR99-19.pdf, fetched 2026-10-26 (unofficial)
[S22] PDF Volume illustration: nonphotorealistic rendering of volume models ..., https://www.csee.umbc.edu/~rheingan/pubs/volillus.tvcg01.pdf, fetched 2026-10-27
[S23] PDF CS 519: Scientific Visualization - University of Illinois Urbana-Champaign, https://courses.grainger.illinois.edu/cs519/fa2017/Lecture%2012%20-%20Volume%20Visualization.pdf, fetched 2026-10-27 (unofficial)
[S24] AnBenLa/OpenGL-MarchingCubes-MarchingTetrahedra-GeometryShader, https://github.com/AnBenLa/OpenGL-MarchingCubes-MarchingTetrahedra-GeometryShader, fetched 2026-10-27 (unofficial)
[S25] Loveof1ife7/cuVRender-A-CUDA-Accelerated-Volume-Rendering-Engine, https://github.com/Loveof1ife7/cuVRender-A-CUDA-Accelerated-Volume-Rendering-Engine, fetched 2026-10-27 (unofficial)
[S26] Rabidus4k/UnityVoxelTerrain, https://github.com/Rabidus4k/UnityVoxelTerrain, fetched 2026-10-28 (unofficial)
[S27] GitHub - yuehaowang/volume_renderer: CUDA-based interactive volume ..., https://github.com/yuehaowang/volume_renderer, fetched 2026-10-29 (unofficial)
[S28] Chapter 7 - Advanced Computer Graphics - VTK, https://examples.vtk.org/site/VTKBook/07Chapter7/, fetched 2026-10-29 (book)
[S29] Implementation of Importance-Driven Volume Rendering, https://www.cg.tuwien.ac.at/research/publications/2005/Thurner-2005-IVR/Thurner-2005-IVR-paper.pdf, fetched 2026-10-29 (unofficial)
[S30] GPU Gems ch39, Volume rendering techniques, https://developer.nvidia.com/sites/all/modules/custom/gpugems/books/GPUGems/gpugems_ch39.html, fetched 2026-10-29
[S31] 3D Isosurfacing, https://doc.cgal.org/latest/Isosurfacing_3/index.html, fetched 2026-10-29
[S32] nerima-games/mc-meshing, https://github.com/nerima-games/mc-meshing, fetched 2026-10-28 (unofficial)
[S33] DCCVT: Differentiable Clipped Centroidal Voronoi Tessellation, https://arxiv.org/abs/2601.13603v1, fetched 2026-10-26
[S34] A hardware architecture for video rate smooth shading of volume data, https://doi.org/10.1016/0097-8493(95)00048-8, fetched 2026-10-29
[S35] Overview of Volume Rendering - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
[S36] Interactive, in-browser cinematic volume rendering of medical images, https://doi.org/10.1080/21681163.2022.2145239, fetched 2026-10-26
