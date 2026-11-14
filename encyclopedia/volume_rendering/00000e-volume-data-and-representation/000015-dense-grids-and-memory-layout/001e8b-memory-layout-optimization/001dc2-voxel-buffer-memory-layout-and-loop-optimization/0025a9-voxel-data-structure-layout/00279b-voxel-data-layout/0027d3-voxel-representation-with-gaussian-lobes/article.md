# Voxel Representation with Gaussian Lobes

Also called: Isotropic Gaussian Voxel Data

## Overview
Voxel Representation with Gaussian Lobes is a compact data structure for storing directional and material information within a voxel grid, designed to facilitate efficient cone tracing for indirect illumination. It solves the problem of storing high-dimensional directional data (normals, light directions) and occlusion metrics in a memory-constrained environment by approximating distributions with isotropic Gaussian lobes. Unlike sibling methods that may store raw per-voxel vectors or complex hierarchical trees without directional abstraction, this method encodes directionality and spread into a single compact voxel footprint, enabling fast interpolation and visibility estimation during rendering.

## How it works
The method represents the directional behavior of a voxel's contents using an isotropic Gaussian distribution characterized by an average direction vector and a standard deviation. It also stores scalar occlusion and material properties.

**Inputs and Outputs**
*   **Input:** A voxel grid (e.g., from a VDB tree) where each leaf node or voxel requires directional and material summarization.
*   **Output:** A dense voxel buffer where each voxel contains a fixed-size data structure.

**Data Structure Layout**
Each voxel stores the following fields in a fixed layout:
1.  **Average Vector $D$**: A 3D vector representing the mean direction (e.g., normal or light direction).
2.  **Standard Deviation $\sigma$**: Encoded implicitly via the norm of $D$.
3.  **Visibility**: A single scalar value representing the percentage of blocked rays (occlusion).
4.  **Material Color**: An RGB vector, pre-multiplied by opacity.
5.  **Normal**: A 3D vector, pre-multiplied by opacity.

**Formulas and Parameters**
The variance $\sigma^2$ is derived from the norm of the average vector $|D|$ to ease interpolation during cone tracing. The relationship is defined as:

$$
\sigma^2 = \frac{1 - |D|}{|D|}
$$

Where:
*   $|D|$ is the Euclidean norm of the average vector $D$.
*   $\sigma$ is the standard deviation of the isotropic Gaussian lobe.

This encoding ensures that as $|D|$ approaches 1 (highly directional), $\sigma$ approaches 0, and as $|D|$ approaches 0 (isotropic), $\sigma$ increases.

**Algorithm Steps**
1.  **Accumulate Directions**: For a given voxel (or leaf node), collect all relevant directional vectors (e.g., surface normals, incoming light directions) from the underlying geometry or data.
2.  **Compute Average Vector $D$**: Calculate the mean of these vectors.
    $$
    D = \frac{1}{N} \sum_{i=1}^{N} v_i
    $$
    where $v_i$ are the individual direction vectors and $N$ is the count.
3.  **Encode Variance**: Compute $|D|$ and derive $\sigma^2$ using the formula above. The value $\sigma$ is not stored explicitly but can be reconstructed during rendering if needed, or the value $|D|$ is stored directly.
4.  **Compute Visibility**: Estimate the occlusion by tracing rays or analyzing the geometry within the voxel to determine the percentage of blocked paths. Store this as a single scalar value.
5.  **Compute Material Color**: Sum the color contributions of all materials within the voxel, weighted by their opacity. Store the result as an alpha-premultiplied RGB value.
6.  **Compute Normal**: Sum the normal vectors weighted by their respective opacities. Store the result as an opacity-premultiplied normal vector.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_gaussian_lobe_voxel(voxel_data):
    """
    Computes the Gaussian lobe representation for a single voxel.
    voxel_data: List of tuples (normal, color, opacity) for all primitives in the voxel.
    Returns: dict with keys 'D', 'visibility', 'color', 'normal'
    """
    N = len(voxel_data)
    if N == 0:
        return {'D': [0, 0, 0], 'visibility': 1.0, 'color': [0, 0, 0], 'normal': [0, 0, 0]}

    # 1. Accumulate Directional Information (Normals)
    sum_normal = [0.0, 0.0, 0.0]
    sum_color = [0.0, 0.0, 0.0]
    total_opacity = 0.0

    for normal, color, opacity in voxel_data:
        # Pre-multiply normal by opacity
        sum_normal[0] += normal[0] * opacity
        sum_normal[1] += normal[1] * opacity
        sum_normal[2] += normal[2] * opacity
        
        # Pre-multiply color by opacity
        sum_color[0] += color[0] * opacity
        sum_color[1] += color[1] * opacity
        sum_color[2] += color[2] * opacity
        
        total_opacity += opacity

    # 2. Compute Average Vector D (from normals)
    # Note: The source uses D for direction. Here we use the accumulated normal as D.
    # In practice, D might be a separate light direction accumulator, but normals are primary.
    D = sum_normal
    
    # 3. Compute Norm |D|
    norm_D = sqrt(D[0]**2 + D[1]**2 + D[2]**2)
    
    # 4. Encode Variance (implicit)
    # sigma^2 = (1 - |D|) / |D|
    # This is used during rendering, not stored directly.
    
    # 5. Compute Visibility
    # Estimate as 1.0 if total_opacity is high, or based on ray casting.
    # Source mentions "single average value".
    visibility = 1.0 - (total_opacity / N) # Simplified heuristic [G]
    # Note: Actual visibility requires ray tracing or density estimation [S1]
    
    # 6. Return Data
    return {
        'D': D,
        'visibility': visibility,
        'color': sum_color,
        'normal': sum_normal
    }
```

**Edge Cases and Pitfalls**
*   **Degenerate Input**: If $|D| = 0$ (isotropic distribution), the variance formula involves division by zero. In practice, a small epsilon or a fallback isotropic value should be used [G].
*   **Precision**: The encoding of $\sigma$ via $|D|$ can lose precision for very small variances.
*   **View Dependency**: Storing a single average visibility value lacks view-dependency, which can lead to inaccuracies for large thin objects [S1].

## Subtopics

### [Leaf Node Gaussian Approximation](14713-leaf-node-gaussian-approximation.md)
Approximates a VDB leaf node (512 voxels) with a single Gaussian primitive. It serves as the lowest LOD, treating the entire leaf as one cluster.

### [Medium LOD Approximation](4128-voxel-representation-with-gaussian-lobes.md)
Subdivides leaf nodes into smaller blocks for better spatial coherence.

### [High LOD Approximation](4128-voxel-representation-with-gaussian-lobes.md)
Further subdivision for higher accuracy at the cost of memory.

## Operating characteristics
*   **Time Complexity**: $O(1)$ per voxel for storage; $O(N)$ for preprocessing where $N$ is the number of primitives in the voxel.
*   **Memory Complexity**: Fixed size per voxel (5 fields). Significantly more compact than storing full directional distributions.
*   **Strengths**: Fast interpolation, low memory footprint, suitable for real-time cone tracing.
*   **Limitations**: Loss of view-dependency in visibility; approximation errors for complex directional distributions; division by zero risk if $|D|=0$.
*   **When to Use**: Real-time indirect illumination with cone tracing; scenes with dense geometry where memory bandwidth is a bottleneck.
*   **When Not to Use**: Scenes requiring high-fidelity view-dependent effects; thin structures where single-value visibility fails.

## Background
The method originates from the paper "Interactive Indirect Illumination Using Voxel Cone Tracing" by N. Schied et al. (2011) [S1]. It builds on earlier work by Hillaire et al. [HSRG07] and Tokuyoshi [Tok05] regarding directional distributions and variance encoding. The technique was developed to enable efficient real-time global illumination by reducing the complexity of directional data in voxel grids.

## Sources
[S1] Interactive Indirect Illumination Using Voxel Cone Tracing, https://research.nvidia.com/sites/default/files/publications/GIVoxels-pg2011-authors.pdf, fetched 2026-10-27
