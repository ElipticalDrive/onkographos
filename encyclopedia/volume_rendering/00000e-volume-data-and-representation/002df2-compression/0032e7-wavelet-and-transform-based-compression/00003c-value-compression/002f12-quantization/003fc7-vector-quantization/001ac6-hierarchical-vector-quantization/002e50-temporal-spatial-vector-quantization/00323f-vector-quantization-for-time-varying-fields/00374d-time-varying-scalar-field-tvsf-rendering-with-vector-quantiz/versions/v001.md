# Time-Varying Scalar Field (TVSF) Rendering with Vector Quantization (HAVS)

Also called: HAVS with VQ, Hardware-Assisted Visibility Ordering for TVSF, TVSF HAVS, Time-Varying HAVS

## Overview
Time-Varying Scalar Field (TVSF) Rendering with Vector Quantization (HAVS) is a GPU-based volume rendering method for time-varying scalar fields defined on unstructured tetrahedral meshes [S1, unofficial]. It solves the problem of rendering large, dynamic datasets without rendering stalls by combining Hardware-Assisted Visibility Sorting (HAVS) with Vector Quantization (VQ) compression [S1, unofficial]. Unlike static rendering, which assumes fixed data, this method uses a paging mechanism to handle arbitrary time steps by swapping codebooks in GPU memory [S1, unofficial]. It sets itself apart from its sibling [Vector Quantization for Time-Varying Fields](6719-vector-quantization-for-time-varying-fields.md) by specifically integrating the HAVS sorting algorithm and pre-integrated volume integrals for efficient compositing on unstructured grids [S1, unofficial].

## How it works
### Inputs and Outputs
*   **Input:** An unstructured tetrahedral mesh with time-varying scalar values. The mesh consists of vertices and tetrahedra.
*   **Output:** A rendered 2D image with correct visibility and shading for the current time step.

### Data Structures
The method relies on specific storage layouts for compressed data and GPU memory.

**1. Compressed Storage (CPU Memory)**
Scalar data is stored using VQ. Each vertex stores:
*   A mean value ($V_0$).
*   Two codebook indices: $i_8$ and $i_{64}$.
*   Total storage per vertex per codebook: 3 values (mean + 2 indices).

**2. Codebook Structure (GPU Texture Memory)**
Codebooks are stored as textures. Each codebook handles 64 time instances.
*   Size per codebook: $72 \text{ KB}$.
*   Composition: $256 \times 64 \times 4\text{B} + 256 \times 8 \times 4\text{B}$.
    *   This implies a structure of $256 \times 64$ entries for one part and $256 \times 8$ for another, likely representing different quantization levels or residual data, though the exact semantic mapping of the two parts is not detailed beyond the size calculation [S1, unofficial].

**3. Tetrahedron Data (GPU Texture Memory)**
To perform ray casting and gradient calculation, each tetrahedron stores:
*   Vertex positions.
*   Gradient matrix (used to calculate the scalar field gradient) [S1, unofficial].
*   Normals are **not** stored; they are reconstructed from vertex positions to save GPU memory [S1, unofficial].

**4. k-buffer (GPU Framebuffer)**
A data structure keeping a fixed number of fragments ($k$) in each pixel for image-space sorting [S1, unofficial].

**5. Lookup Table**
Contains pre-integrated volume integrals used to determine color and opacity for fragments [S1, unofficial].

### Numbered Steps

1.  **Initialization and Paging Setup:**
    *   Calculate the number of codebooks $c = t / 64$, where $t$ is the total number of time instances [S1, unofficial].
    *   Load the first two codebooks into GPU texture memory when rendering starts [S1, unofficial].

2.  **CPU Decompression (Per Rendering Pass):**
    *   For each vertex in the mesh, retrieve the mean $V_0$ and indices $i_8, i_{64}$ from CPU memory [S1, unofficial].
    *   Perform decompression to reconstruct scalar values.
    *   Pass these reconstructed scalar values as texture coordinates to the GPU [S1, unofficial].
    *   *Note:* Previous attempts to reconstruct in the Vertex Shader were too slow due to limited operations and texture fetch latency. Passing changing per-face constants to the Fragment Shader was also deemed slow [S1, unofficial].

3.  **Face Sorting (CPU):**
    *   Sort mesh faces by their centroids in object-space [S1, unofficial]. This provides a partial order.

4.  **Rasterization and k-Buffer Sorting (GPU):**
    *   Rasterize the sorted faces.
    *   For each new fragment rasterized, compare it with entries in the k-buffer [S1, unofficial].
    *   Identify the two entries closest to the viewpoint (front-to-back) [S1, unofficial].

5.  **Scalar Reconstruction (Fragment Shader):**
    *   For each vertex of the current face, recover the mean and indices $i_8, i_{64}$ (passed as texture coordinates or constants) [S1, unofficial].
    *   Use $i_8$ and $i_{64}$ as the $v$ coordinate to access the codebook textures.
    *   Use the current time step $t$ as the $u$ coordinate for both codebooks.
    *   Reconstruct four scalar values from the codebook lookups [S1, unofficial].

6.  **Gradient and Value Computation:**
    *   Calculate the gradient of the scalar field using the gradient matrix stored with the tetrahedron data [S1, unofficial].
    *   Compute the scalar value for the entry and exit points of the ray within the tetrahedron using the algorithm from [7] (Lürig et al., Implicit Adaptive Volume Ray-Casting) [S1, unofficial].

7.  **Compositing:**
    *   Use the lookup table containing pre-integrated volume integrals to find color and opacity for the two closest fragments [S1, unofficial].
    *   Composite the color and opacity into the framebuffer.
    *   Write the remaining fragments back to the k-buffer [S1, unofficial].

8.  **Codebook Swapping (Paging):**
    *   After accessing the last time instance in the first codebook, swap texture references to the second set (already in memory) [S1, unofficial].
    *   Simultaneously load the next codebook into the memory location of the first one [S1, unofficial]. This avoids pipeline stalls [S1, unofficial].

### Formulas

**Compressed Storage Size:**
$$ \text{size}_{VQ} = v \times c \times 3 \times 4\text{B} + c \times 72\text{KB} $$
Where:
*   $v$: Number of vertices.
*   $c$: Number of codebooks ($c = t/64$).
*   $3$: Mean plus two indices ($i_8, i_{64}$).
*   $4\text{B}$: Size of a float/integer value.
*   $72\text{KB}$: Size of one codebook ($256 \times 64 \times 4\text{B} + 256 \times 8 \times 4\text{B}$) [S1, unofficial].

**Uncompressed Storage Size:**
$$ \text{size}_u = v \times t \times 4\text{B} $$
Where:
*   $t$: Number of time instances.
*   $4\text{B}$: Size of a float [S1, unofficial].

### Parameters
*   **Codebook Capacity:** 64 time instances per codebook [S1, unofficial].
*   **Scalar Storage Format:** Gradient of the scalar field and a reference static value allow reconstruction via interpolation [S1, unofficial].
*   **Hardware:** Tested on GeForce 6800GT [S1, unofficial].

### Edge Cases and Pitfalls
*   **Vertex Shader Reconstruction:** Reconstructing scalar values in the Vertex Shader using texture access was found to be slow due to limited operations and latency [S1, unofficial].
*   **Fragment Shader Constants:** Sending changing per-face constant information (means/indices) to the fragment shader as program arguments is slow if changed for each face [S1, unofficial].
*   **Non-Convex Meshes:** The method uses a depth-peeling approach to handle non-convex meshes, which is more general than the method in [13] (Weiler et al.) [S1, unofficial].
*   **Complex Scalar Fields:** Datasets with complex, aleatory scalar field motion (e.g., Brain dataset) result in lower SNR and higher error [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Rendering rates are nearly the same as static rendering for most datasets. On the largest dataset (SPX2, 808K tetrahedra), it is only 40% slower than static rendering [S1, unofficial].
*   **Memory Complexity:** Compressed size is significantly smaller than uncompressed.
    *   Formula: $\text{size}_{VQ} = v \times c \times 3 \times 4\text{B} + c \times 72\text{KB}$ [S1, unofficial].
    *   GPU memory usage is optimized by removing normals and using a paging mechanism for codebooks [S1, unofficial].
*   **Compression Ratio:** Ranges from 11.43 (TORSO) to 20.57 (SPX2) [S1, unofficial].
*   **Quality (SNR):** Minimum SNR ranges from 2.96 (BRAIN) to 41.70 (BLUNT) [S1, unofficial].
*   **Strengths:** Handles arbitrary time steps without stalls; efficient for unstructured meshes; interactive rates for most datasets.
*   **Limitations:** Performance degrades on very large datasets (40% slower); compression quality varies with data complexity (low SNR for complex fields) [S1, unofficial].
*   **When to use:** Interactive visualization of time-varying scalar fields on unstructured grids where bandwidth or memory is constrained.
*   **When not to use:** Static data (use standard HAVS); highly complex scalar fields requiring high fidelity (low SNR) [S1, unofficial].

## Background
*   **Origin:** Proposed in [2] (referenced in [S1, unofficial]), which introduced Hardware-Assisted Visibility Sorting (HAVS). The extension to TVSF with VQ is detailed in [S1, unofficial].
*   **History:** Developed to close the gap between rendering time-varying data on unstructured and structured grids [S1, unofficial].
*   **Usage:** Used in volume visualization of dynamic data, such as rotating dipoles (Torso) and EEG readings (Brain) [S1, unofficial].

## Sources
[S1] Volume Rendering of Time-Varying Scalar Fields on Unstructured Meshes, https://www.sci.utah.edu/~stevec/papers/tvrc.pdf, fetched 2026-10-27 (unofficial)
