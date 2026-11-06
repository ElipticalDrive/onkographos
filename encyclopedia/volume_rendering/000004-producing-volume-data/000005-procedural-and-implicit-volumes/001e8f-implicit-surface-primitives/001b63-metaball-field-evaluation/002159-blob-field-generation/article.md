# Blob Field Generation

Also called: Inverse Distance Weighting for Isosurfaces, Inverse Square Blob Field, Metaball Scalar Field Construction, Procedural Blob Field

## Overview
Blob Field Generation is a procedural method for constructing a 3D scalar field by summing inverse-squared Euclidean distance contributions from a set of source points (metaballs) [S1, S2, unofficial]. It solves the problem of defining smooth, organic implicit surfaces for isosurface extraction algorithms like Marching Cubes [S1, S2, unofficial]. It is distinguished from sibling methods by its specific use of the inverse square falloff ($1/r^2$) rather than Gaussian decay or fuzzy hyper-parameter blending, resulting in sharper, more defined "blob" interactions [S1, S2, unofficial].

## How it works
**Inputs:**
*   A set of $N$ source points (metaball centers) $\mathbf{P}_i = (x_i, y_i, z_i)$ for $i \in [0, N-1]$ [S2, unofficial].
*   A set of weights $w_i$ for each source point [S2, unofficial].
*   An evaluation point $\mathbf{V} = (x, y, z)$ [S2, unofficial].
*   (Optional) A time parameter $t$ if source positions are animated [S1, unofficial].
*   (Optional) A floor effect parameter if a ground plane is included [S1, unofficial].

**Outputs:**
*   A scalar field value $f(\mathbf{V})$ representing the density at $\mathbf{V}$ [S2, unofficial].

**Data Structures:**
*   `SourcePoint`: A structure or tuple containing three floating-point coordinates $(x, y, z)$ [S2, unofficial].
*   `Weights`: An array of floating-point values corresponding to each source point [S2, unofficial].

**Formulas:**
The core field function sums the weighted inverse squared distances from the evaluation point to each source point [S2, unofficial]:

$$ f(\mathbf{V}) = \sum_{i=0}^{N-1} \frac{w_i}{d_i^2} $$

Where $d_i^2$ is the squared Euclidean distance between $\mathbf{V}$ and $\mathbf{P}_i$:

$$ d_i^2 = (x - x_i)^2 + (y - y_i)^2 + (z - z_i)^2 $$

If a floor effect is applied, a constant source point (e.g., at $y = -\infty$ or a specific $y_{floor}$) is added to the sum with its own weight and distance calculation [S1, unofficial].

**Parameters:**
*   $w_i$: Weight of the $i$-th blob. In the reference implementation, weights are $0.5, 1.0, 1.5$ for three blobs [S2, unofficial]. In another implementation, a uniform strength is calculated as $strength = 1.2 / ((\sqrt{numBlobs} - 1) / 4 + 1)$ [S1, unofficial].
*   $N$: Number of blobs. Typical examples use $N=3$ [S2, unofficial].
*   Isosurface Threshold: The value $T$ at which the surface is extracted. The sources do not specify a universal $T$, but note that the field value approaches infinity near source points and zero far away [S2, unofficial].

**Steps:**
1.  Initialize the accumulator `fieldValue` to $0.0$ [S2, unofficial].
2.  For each source point $i$ from $0$ to $N-1$:
    a. Calculate the difference vector components:
       $$ dx = x - x_i $$
       $$ dy = y - y_i $$
       $$ dz = z - z_i $$
    b. Calculate the squared distance:
       $$ distSq = dx^2 + dy^2 + dz^2 $$
    c. Add the weighted inverse squared distance to the accumulator. To prevent division by zero, add a small epsilon $\epsilon$ to the denominator [S1, unofficial]:
       $$ fieldValue \leftarrow fieldValue + \frac{w_i}{distSq + \epsilon} $$
       *Note: The reference code uses $\epsilon = 0.0$ for the primary blobs but implies handling via logic or assumes non-coincident points, while the WebGL implementation explicitly uses $\epsilon = 0.000001$ [S1, S2, unofficial].*
3.  (Optional) If a floor effect is desired:
    a. Calculate distance to the floor plane (e.g., $y_{floor}$). Let $fy = y - y_{floor}$.
    b. Calculate floor value:
       $$ floorVal = \frac{w_{floor}}{fy^2 + \epsilon_{floor}} - C $$
       Where $w_{floor}$ is the floor weight (e.g., $2.0$) and $C$ is a constant offset (e.g., $12.0$) [S1, unofficial].
    c. If $floorVal > 0$, add it to `fieldValue` [S1, unofficial].
4.  Return `fieldValue`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def evaluate_blob_field(x, y, z, source_points, weights, epsilon=1e-6, floor_y=None, floor_weight=0.0, floor_offset=0.0):
    """
    Evaluates the scalar field value at point (x, y, z) based on inverse squared distance to source points.
    
    Args:
        x, y, z: Coordinates of the evaluation point.
        source_points: List of tuples (px, py, pz) representing blob centers.
        weights: List of floats representing the weight of each blob.
        epsilon: Small constant to prevent division by zero.
        floor_y: If not None, adds a floor effect at this y-coordinate.
        floor_weight: Weight for the floor effect.
        floor_offset: Constant subtracted from the floor contribution.
        
    Returns:
        float: The scalar field value.
    """
    field_value = 0.0
    
    # Sum contributions from metaballs
    for i in range(len(source_points)):
        px, py, pz = source_points[i]
        w = weights[i]
        
        dx = x - px
        dy = y - py
        dz = z - pz
        
        dist_sq = dx*dx + dy*dy + dz*dz
        
        # Add inverse squared distance weighted by blob strength
        # Using epsilon to avoid division by zero if point coincides with source
        field_value += w / (dist_sq + epsilon)
        
    # Optional floor effect
    if floor_y is not None:
        fy = y - floor_y
        floor_val = floor_weight / (fy*fy + epsilon) - floor_offset
        if floor_val > 0:
            field_value += floor_val
            
    return field_value
```

**Edge Cases and Pitfalls:**
*   **Division by Zero:** If the evaluation point $\mathbf{V}$ coincides exactly with a source point $\mathbf{P}_i$, the distance squared is zero, causing the term to approach infinity [S2, unofficial]. A small epsilon must be added to the denominator [S1, unofficial].
*   **Metric Choice:** Standard Euclidean distance produces spherical blobs. To create cubic ("Minecraft-like") isosurfaces, alternative metrics like Taxicab (Manhattan) or Max-metric must replace the Euclidean distance calculation [S2, unofficial].
*   **Precision:** The field values can become very large near source points. Floating-point precision limits may affect the smoothness of the isosurface extraction near these singularities.

## Subtopics

### Radial Gaussian Density Field Generation
Uses Gaussian falloff ($e^{-r^2}$) instead of inverse square. Results in smoother, softer transitions between blobs compared to the sharper cutoff of inverse square fields. Choose when organic, less defined blending is desired. [Link](2081-radial-gaussian-density-field-generation.md)

### Fuzzy Metaballs Hyper-parameters
Adjusts the blending behavior and isovalue thresholds dynamically. Choose when fine-tuning the "stickiness" or separation of blobs is required beyond simple weight adjustments. [Link](8382-fuzzy-metaballs-hyper-parameters.md)

## Operating characteristics
*   **Time Complexity**: $O(N)$ per evaluation point, where $N$ is the number of source points. For a grid of $V$ voxels, the total complexity is $O(N \cdot V)$.
*   **Memory Complexity**: $O(N)$ to store source points and weights. The field itself requires $O(V)$ storage if precomputed, or $O(1)$ if evaluated on-the-fly.
*   **Performance**: The method is computationally inexpensive per voxel, involving only basic arithmetic operations. It scales linearly with the number of blobs.
*   **Strengths**: Simple to implement; produces well-defined, sharp isosurfaces; easy to animate by updating source positions.
*   **Limitations**: Singularities at source points require epsilon handling; inverse square falloff decays slower than Gaussian, potentially requiring larger bounding boxes or higher isovalue thresholds to clip distant noise.
*   **When to use**: When sharp, distinct blob shapes are desired, or when implementing classic metaball effects for Marching Cubes.
*   **When not to use**: When smooth, diffuse blending is preferred (use Gaussian) or when computational cost of $O(N)$ per voxel is prohibitive for very large $N$ (consider spatial partitioning or GPU acceleration).

## Background
The method originates from the classic "metaball" concept introduced by Wertheimer in 1986, where scalar fields are summed to create organic shapes [S1, S2, unofficial]. The specific inverse square implementation is widely cited in early graphics tutorials and online resources, such as Paul Bourke's Marching Cubes implementation and subsequent Stack Overflow discussions analyzing it [S2, unofficial]. It serves as a foundational technique for procedural implicit surface generation in real-time graphics and VFX.

## Sources
[S1] 3merillon/WebGL2-GPU-Marching-Cubes, https://github.com/3merillon/WebGL2-GPU-Marching-Cubes, fetched 2026-10-28 (unofficial)
[S2] 2 Isovalue in Marching Cubes Algorithm, https://stackoverflow.com/questions/11177604/isovalue-in-marching-cubes-algorithm?rq=4, fetched 2026-10-30 (unofficial)
