# Virtual ray lights (VRLs)

Also called: Novak et al. method, VRL integration

## Overview
Virtual ray lights (VRLs) is a many-light rendering method for participating media that reinterprets light transport accumulation as a bidirectional segment-segment connection technique [S1]. It traces rays from light sources, stores the first ray segment after a surface interaction as a "Virtual Ray Light," and organizes these segments into a grid hierarchy to efficiently compute multiple scattering contributions to camera rays using precomputed tables [S2]. Unlike standard path tracing which samples paths stochastically, VRLs use a deterministic lookup based on geometric relationships between light segments and eye segments, trading memory for speed in dense media [S2].

## How it works
### Inputs and Outputs
**Inputs:**
*   $S$: Camera segment ray defined by origin $P$ and direction $\omega_t$ [S2].
*   $G$: Grid hierarchy containing all VRLs [S2].
*   $A$: Accumulated segment table (precomputed radiance data) [S2].
*   $\epsilon$: Solid angle threshold for culling [S2].
*   $\ell$: Mean-free path of the medium [S2].

**Outputs:**
*   $R$: Multiple scattering radiance contribution [S2].

### Data Structures
**VRL Attributes [S2]:**
Each Virtual Ray Light stores:
1.  Position of ray origin ($v$) [S2].
2.  Ray direction ($d$) [S2].
3.  Light ray length before the next surface event [S2].
4.  Radiance at the ray origin [S2].

**Grid Hierarchy [S2]:**
*   Structure: Two to five levels of regular grids. Level $i+1$ is computed from level $i$ by merging eight grid cells together [S2].
*   Cell Content:
    *   Index of all VRLs inside the cell [S2].
    *   A representative VRL [S2].
    *   Bounding box [S2].
*   Storage: VRLs are stored in a 1D buffer, reordered for continuous storage per cell [S2].

### Formulas
**Cylindrical Coordinates [S2]:**
To map a camera ray point $P$ relative to a VRL axis defined by origin $v$ and direction $d$:
$$ z = (P - v) \cdot d $$
$$ \rho = \|(P - v) - z d\| $$
where $z$ is the axial distance and $\rho$ is the radial distance from the VRL axis [S2].

**Weight Factor [S2]:**
The weight $W$ accounts for the ratio of the camera segment length to the accumulated segment length (defined as $5 \times$ mean-free path, $\ell$):
$$ W = \frac{\sum_{i=1}^k e^{-i\ell}}{\sum_{i=1}^5 e^{-i\ell}} $$
where $k$ is the length of the camera segment divided by $5 \times \ell$ [S2].

**Radiance Reconstruction [S2]:**
The multiple scattering contribution is computed using a precomputed table $\hat{\Phi}$:
$$ \text{mult}(P, \omega_t) = W \hat{\Phi}\left(\frac{\rho}{\ell}, \frac{z}{\ell}, T_{(v,d)}(\omega_t)\right) $$
where $T_{(v,d)}(\omega_t)$ is the direction transformation of $\omega_t$ into the frame defined by $(v, d)$ [S2].

### Algorithm
The rendering loop iterates over grid cells and VRLs to accumulate radiance [S2]:

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def dot(a, b):
    return sum(x * y for x, y in zip(a, b))

def norm(v):
    return math.sqrt(sum(x * x for x in v))

def compute_vrl_radiance(camera_origin, camera_direction, camera_length, grid_cells, segment_table, epsilon, mfp):
    """
    Computes multiple scattering radiance for a camera segment using VRLs.
    
    Args:
        camera_origin: 3D vector P
        camera_direction: 3D vector omega_t
        camera_length: float length of camera segment
        grid_cells: List of grid cells in the hierarchy
        segment_table: Precomputed table A of radiance values
        epsilon: Solid angle threshold for culling
        mfp: Mean-free path (l) of the medium
        
    Returns:
        R: Accumulated radiance.
    """
    R = 0.0
    
    # Precompute denominator for W
    denom = sum(math.exp(-i * mfp) for i in range(1, 6))
    
    for cell in grid_cells:
        # Discard grid cells with small contribution
        # solid_angle is a standard geometric function computing the solid angle 
        # subtended by the cell's bounding box at the camera origin [S2]
        if solid_angle(cell.bounding_box, camera_origin) <= epsilon:
            continue
            
        vrls = cell.vrls # List of VRLs in this cell
        
        for vrl in vrls:
            # Compute relative coordinate between vrl and camera segment
            diff = (camera_origin[0] - vrl.origin[0], 
                    camera_origin[1] - vrl.origin[1], 
                    camera_origin[2] - vrl.origin[2])
            z = dot(diff, vrl.direction)
            rho = norm((diff[0] - z * vrl.direction[0], 
                        diff[1] - z * vrl.direction[1], 
                        diff[2] - z * vrl.direction[2]))
            
            # Determine index in the accumulated segment table
            # relativeCoordinate maps normalized rho, z, and transformed direction to table indices [S2]
            index = relativeCoordinate(vrl, camera_direction)
            
            # Calculate k based on camera segment length and mean-free path
            k = camera_length / (5.0 * mfp)
            
            # Calculate W
            num = sum(math.exp(-i * mfp) for i in range(1, int(k) + 1))
            W = num / denom
            
            # Add contribution
            # power(vrl) is the radiance at the ray origin
            # weight(S) is effectively the weight W and table lookup
            R += segment_table[index] * vrl.radiance * W
            
    return R
```

### Parameters
*   **Solid angle threshold $\epsilon$**: Used to discard grid cells with small contribution [S2].
*   **Accumulated segment length**: Defined as $5 \times$ mean-free path ($\ell$) [S2].

### Edge Cases and Pitfalls
*   **Precision**: The exponential terms in $W$ can underflow for large $i$ or small $\ell$ [G]. Implementation should use log-sum-exp or careful ordering to maintain numerical stability [G].
*   **Boundaries**: The grid hierarchy must handle VRLs near cell boundaries correctly [S2]. The 1D buffer reordering ensures continuous storage per cell [S2].

## Subtopics

### Joint Path Vertex Sampling
The parent method that VRLs reinterpret as a bidirectional segment-segment connection technique [S1]. VRLs provide a specific efficient implementation for the segment-segment connection in participating media [S1]. [Link](2136-joint-path-vertex-sampling.md)

### Bridge Sampling
A sibling method for connecting subpaths [S2]. VRLs differ by using precomputed tables and grid hierarchies rather than direct PDF cancellation for all terms [S1, S2]. [Link](3881-bridge-sampling.md)

### Bidirectional Path Tracing
A sibling method [S2]. VRLs are more specialized for participating media and use precomputation, whereas BPT is a general-purpose unbiased method [G]. [Link](6666-bidirectional-path-tracing.md)

## Operating characteristics
*   **Time Complexity**: Not given in the sources.
*   **Memory Complexity**: Dominated by the grid hierarchy and the precomputed segment table [S2]. The table size depends on the resolution of $\rho$, $z$, and angular dimensions [S2].
*   **Performance**: Designed for interactive rendering on GPU using Optix [S2]. The grid hierarchy and culling reduce the number of VRLs processed [S2].
*   **Strengths**: Efficient for scenes with many light sources and participating media [S2]. Precomputation allows fast runtime evaluation [G].
*   **Limitations**: High memory usage for the precomputed tables [S2]. Limited to specific scattering models stored in the table [S2]. Not suitable for dynamic media where the scattering properties change rapidly [G].
*   **When to use**: Interactive rendering of participating media with static lighting and material properties [S2].
*   **When not to use**: Dynamic media, highly complex scattering models not covered by the precomputed table, or scenes with very few lights where direct path tracing is sufficient [G].

## Background
Virtual ray lights were originally developed by Novak et al. in 2012 [S1]. S2 describes a rendering algorithm based on VRLs implemented on the GPU using Optix, utilizing a neural network model for scattering tables [S2]. It is used in computer graphics for rendering participating media such as fog, smoke, and clouds [S2].

## Sources
[S1] Joint importance sampling of low-order volumetric scattering, https://doi.org/10.1145/2508363.2508411, fetched 2026-10-28
[S2] Interactive Simulation of Scattering Effects in Participating Media Using a Neural Network Model, https://doi.org/10.1109/TVCG.2019.2963015, fetched 2026-10-26
