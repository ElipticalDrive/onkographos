# Voxel Saliency

Also called: Saliency map

## Overview
Voxel Saliency is a feature importance metric for volume data defined as the magnitude of the gradient of the obscurance field [S1]. It identifies regions of high perceptual importance, such as edges, occlusion variations, and corners, by measuring how rapidly obscurance changes across a local neighborhood [S1]. Unlike sibling methods that may rely on direct intensity gradients or semantic labels, Voxel Saliency is derived from volumetric shadowing information (obscurance), making it particularly effective for highlighting structural boundaries in complex, self-occluding datasets [S1]. It serves as the core importance specification mechanism in the Importance-Driven Visualization Model, enabling viewpoint selection and transfer function modulation [S1].

## How it works
**Inputs:**
*   A 3D volume dataset $V$.
*   The precomputed obscurance field $O$, where $O(k)$ is the obscurance value for voxel $k$ [S1].

**Outputs:**
*   A scalar saliency map $S$, where $S(p) \in [0, 1]$ for each voxel $p$ [S1].

**Data Structures:**
*   **Obscurance Field:** A 3D array of floating-point values, same dimensions as the input volume [S1].
*   **Saliency Map:** A 3D array of floating-point values, same dimensions as the input volume [S1].

**Parameters:**
*   **Neighborhood Size:** A cubic neighborhood of $5^3$ voxels centered at the query voxel $p$ [S1]. (Note: A $3^3$ neighborhood was used in [NCKG00], but $5^3$ yields smoother results [S1]).
*   **Scaling:** Linear scaling to $[0, 1]$ per data volume [S1].

**Algorithm:**
1.  **Compute Obscurance:** Calculate the obscurance $O(k)$ for all voxels $k$ in the volume [S1].
2.  **Iterate Voxels:** For each voxel $p$ in the volume:
    a. Define the neighborhood $\mathcal{N}_p$ as the set of voxels $k$ within a $5^3$ cube centered at $p$ [S1].
    b. Initialize gradient components $A = 0$, $B = 0$, $C = 0$ [S1].
    c. For each voxel $k \in \mathcal{N}_p$:
        i. Calculate the vector components $\vec{d}_{pk} = (x(k), y(k), z(k))$ from $p$ to $k$ [S1].
        ii. Calculate the distance weight $w(k)$ between voxels $p$ and $k$ [S1].
        iii. Accumulate components:
            $$A \leftarrow A + w(k) \cdot O(k) \cdot x(k)$$
            $$B \leftarrow B + w(k) \cdot O(k) \cdot y(k)$$
            $$C \leftarrow C + w(k) \cdot O(k) \cdot z(k)$$
    d. Compute the saliency magnitude:
        $$S(p) = \sqrt{A^2 + B^2 + C^2}$$
3.  **Normalize:** For the specific volume dataset, scale the map to $[0, 1]$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_voxel_saliency(obscurance_field, dims):
    """
    Computes voxel saliency as the magnitude of the obscurance gradient.
    obscurance_field: 3D list of lists of lists of floats, O(k).
    dims: tuple (Dx, Dy, Dz) of the volume dimensions.
    Returns: 3D list of lists of lists of floats, S(p) scaled to [0, 1].
    """
    # Initialize output saliency map
    saliency_map = [[[0.0 for _ in range(dims[2])] for _ in range(dims[1])] for _ in range(dims[0])]
    
    # Precompute neighborhood offsets for a 5x5x5 cube (radius 2)
    offsets = []
    for dx in range(-2, 3):
        for dy in range(-2, 3):
            for dz in range(-2, 3):
                offsets.append((dx, dy, dz))
    
    # Compute raw saliency for each voxel p
    for px in range(dims[0]):
        for py in range(dims[1]):
            for pz in range(dims[2]):
                A = 0.0
                B = 0.0
                C = 0.0
                
                for dx, dy, dz in offsets:
                    kx, ky, kz = px + dx, py + dy, pz + dz
                    
                    # Boundary check
                    if not (0 <= kx < dims[0] and 0 <= ky < dims[1] and 0 <= kz < dims[2]):
                        continue
                        
                    # Vector components from p to k
                    x_k, y_k, z_k = dx, dy, dz
                    
                    # Obscurance at k
                    O_k = obscurance_field[kx][ky][kz]
                    
                    # Weight w(k) is the distance between p and k [S1]
                    w_k = math.sqrt(x_k**2 + y_k**2 + z_k**2)
                    
                    # Accumulate gradient components [S1]
                    A += w_k * O_k * x_k
                    B += w_k * O_k * y_k
                    C += w_k * O_k * z_k
                    
                # S(p) = sqrt(A^2 + B^2 + C^2) [S1]
                saliency_map[px][py][pz] = math.sqrt(A**2 + B**2 + C**2)
                
    # Normalize to [0, 1] per data volume [S1]
    max_saliency = 0.0
    for px in range(dims[0]):
        for py in range(dims[1]):
            for pz in range(dims[2]):
                if saliency_map[px][py][pz] > max_saliency:
                    max_saliency = saliency_map[px][py][pz]
                    
    if max_saliency > 0:
        for px in range(dims[0]):
            for py in range(dims[1]):
                for pz in range(dims[2]):
                    saliency_map[px][py][pz] /= max_saliency
                    
    return saliency_map
```

## Subtopics

### XS Map (Extinction-Saliency Map)
The XS Map is a combined importance map used in perceptual rendering of participating media [S2]. It integrates the Saliency map ($S$) with an Extinction map ($X$) to guide selective rendering [S2]. The combination is defined as $XS(w_x, w_s, op)$, where $w_x$ and $w_s$ are weighting coefficients and $op$ is the combination operator [S2]. The default configuration uses equal weighting ($w_x=0.5, w_s=0.5$) and an additive operator ($+$) to preserve all weighted features [S2]. A key limitation is that the XS Map fails to identify areas of strong multiple scattering around light sources, leading to poor sampling and visible artifacts in those regions [S2].

## Operating characteristics
*   **Time and Memory Complexity:** Not given in the sources.
*   **Scaling:** The method is scale-dependent; the saliency value depends on the size of the neighborhood considered [S1].
*   **Strengths:** Effectively highlights structural edges, occlusion variations, and corners in volumetric data [S1].
*   **Limitations:** Does not account for light scattering artifacts around sources, which can cause visible artifacts if used alone [S2].
*   **Usage:** Primarily used for automatic viewpoint selection (maximizing visible saliency) and for modulating transfer functions (opacity/color) to enhance salient features [S1].

## Background
Voxel Saliency was proposed in the Obscurance-based Volume Rendering Framework [S1]. It adapts the concept of mesh saliency [LVJ05] to volumetric data [S1]. The gradient estimation technique is based on the 4D linear regression method proposed in [NCKG00] [S1]. The method emphasizes that obscurance, representing occlusion information, varies perceptually at edges and corners, making its gradient a suitable saliency measure [S1].

## Sources
[S1] Obscurance-based Volume Rendering Framework, https://doi.org/10.2312/vg/vg-pbg08/113-120, fetched 2026-10-26
[S2] Perceptual rendering of participating media, https://doi.org/10.1145/1278387.1278389, fetched 2026-10-30
