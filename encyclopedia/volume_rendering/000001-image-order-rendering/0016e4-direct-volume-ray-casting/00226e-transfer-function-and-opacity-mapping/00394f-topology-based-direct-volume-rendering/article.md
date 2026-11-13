# Topology-Based Direct Volume Rendering

Also called: Topology in direct volume rendering

## Overview
Topology-Based Direct Volume Rendering applies topological segmentation to scalar fields to guide direct volume rendering attributes (opacity, color), rather than relying solely on indirect mesh visualization [S1, unofficial]. It solves the problem of visually distinguishing complex, overlapping structures in scalar data by using topological features (such as connectivity and persistence) to modulate rendering properties, allowing outer objects to be rendered with lower opacity to reveal internal features [S1, unofficial]. It sets itself apart from its siblings by deriving rendering weights from global topological structures (like contour trees) rather than local scalar values or gradients alone.

## How it works
**Inputs and Outputs**:
- Input: A scalar volume dataset $V$, a topological segmentation structure (e.g., contour tree or volume skeleton tree) derived from $V$, and a camera configuration.
- Output: A 2D image where pixel colors are determined by direct volume ray casting, with optical properties modulated by topological attributes.

**Data Structures**:
- **Scalar Field**: A 3D grid of scalar values.
- **Topological Structure**: A graph or tree representation of the scalar field's topology. For contour trees, this typically includes nodes representing extrema (minima/maxima) and saddles, and edges representing the evolution of contour components. The structure identifies boundaries between topological objects [S1, unofficial].

**Steps**:
1. **Topological Segmentation**: Compute a topological structure (e.g., contour tree) for the scalar field. Identify boundaries between distinct topological objects or regions [S1, unofficial].
2. **Attribute Assignment**: Assign rendering attributes (opacity $\alpha$, color $\mathbf{c}$) to samples based on the topological attributes of the object to which the sample belongs.
   - *Example*: Takeshima et al. use attributes such as the number of equal-valued contours and occlusion to assign opacity levels [S1, unofficial].
   - *Specific Strategy*: Outermost objects (those with fewer enclosing contours or higher occlusion potential) are assigned lower opacities to prevent obscuring features centered in the volume [S1, unofficial].
3. **Direct Volume Rendering**: Perform standard direct volume ray casting. For each ray, at each sample point $\mathbf{p}_i$:
   - Retrieve the scalar value $V(\mathbf{p}_i)$.
   - Determine the topological object ID for $\mathbf{p}_i$ (from the precomputed structure).
   - Map the topological object ID and/or its attributes to opacity $\alpha_i$ and color $\mathbf{c}_i$. This replaces or augments the standard transfer function $T(V) \rightarrow (\mathbf{c}, \alpha)$ [S1, unofficial].
   - Composite samples using the front-to-back equation [S2789].

**Pseudocode**:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Preprocessing (Offline/Pre-computation)
def compute_topology(volume):
    """
    Computes a contour tree or volume skeleton tree for the scalar volume.
    Returns a structure mapping voxel indices or spatial regions to topological IDs.
    """
    # Implementation depends on specific topology library (e.g., GUDHI, VMTK)
    # Not given in the sources. [G]
    pass

def assign_topological_attributes(topology_structure):
    """
    Assigns opacity and color weights to topological objects.
    Returns a lookup table or function mapping TopologicalID -> (alpha, color).
    """
    # Example logic from Takeshima et al.:
    # Objects with higher occlusion or fewer enclosing contours get lower alpha.
    # Not given in the sources. [G]
    pass

# Rendering (Per Ray)
def render_ray(topology_lookup, topological_weights, ray_origin, ray_direction, volume):
    """
    Renders a single ray using topology-based opacity.
    """
    acc_color = (0.0, 0.0, 0.0)
    acc_alpha = 0.0
    
    # Intersect ray with volume bounding box
    t_near, t_far = intersect_bbox(ray_origin, ray_direction, volume.bounds)
    
    num_samples = 100 # Not given in the sources. [G]
    dt = (t_far - t_near) / num_samples
    
    for i in range(num_samples):
        t = t_near + i * dt
        p = ray_origin + t * ray_direction
        
        # Sample scalar value
        v_val = volume.sample(p) # Trilinear interpolation [S2789]
        
        # Determine topological object ID for this point
        # This requires mapping spatial position p to the precomputed topology
        topo_id = topology_lookup.get_id(p)
        
        # Get topological attributes (opacity, color)
        alpha, color = topological_weights[topo_id]
        
        # Standard front-to-back compositing
        # Note: Alpha correction for step size may still be applied if alpha is extinction-based
        # Here, alpha is directly assigned from topology
        weight = alpha * (1.0 - acc_alpha)
        acc_color += weight * color
        acc_alpha += weight
        
        if acc_alpha >= 1.0:
            break
            
    return acc_color
```

**Parameters**:
- **Topological Simplification Threshold**: Not given in the sources. [G]
- **Opacity Mapping Strategy**: Takeshima et al. use occlusion and contour count [S1, unofficial].

**Edge Cases**:
- **Degenerate Topology**: Flat regions or constant-value areas may not have well-defined topological boundaries.
- **Performance**: Computing contour trees for large volumes is expensive. The topology must be precomputed or simplified.

## Subtopics

### [Hardware-Accelerated Topological Volume Rendering](11664-hardware-accelerated-topological-volume-rendering.md)
Applies topological segmentation to guide rendering attributes by dynamically rendering "tubes" between saddles and extrema in the fragment shader, enabling interactive manipulation of topological simplifications like branch pruning.

## Operating characteristics
- **Time Complexity**: Preprocessing (topology computation) is typically $O(N \log N)$ or worse depending on the algorithm and dimensionality, where $N$ is the number of voxels. Rendering time is similar to standard direct volume rendering, $O(R \cdot S)$, where $R$ is the number of rays and $S$ is samples per ray, plus the cost of topology lookup.
- **Memory Complexity**: Requires storage for the scalar volume and the topological structure (e.g., contour tree). The contour tree size is proportional to the number of critical points, which is generally much smaller than $N$, but the mapping from spatial coordinates to topological IDs may require additional storage.
- **Strengths**: Provides semantic understanding of the data structure; allows for intuitive visual separation of complex overlapping features; reduces visual clutter by modulating opacity based on structural importance.
- **Limitations**: Computationally expensive preprocessing; sensitive to noise in the scalar field (topology changes significantly with small perturbations); requires careful tuning of topological simplification parameters.
- **When to use**: When the user needs to understand the connectivity and hierarchy of features in a scalar field, or when standard transfer functions fail to separate overlapping structures.
- **When not to use**: For real-time rendering of very large datasets without precomputed topology; when the scalar field is noisy and topology is unstable.

## Background
The method was discussed in the context of topological visualization techniques for volume multifield data [S1, unofficial]. Takeshima et al. [44] (cited in [S1, unofficial]) were among the early adopters, using topological attributes like the number of equal-valued contours and occlusion to assign opacity levels in direct volume rendering. The approach contrasts with mesh-based topological visualization (Section 2.5 in [S1, unofficial]) by keeping the rendering direct.

## Sources
[S1] Chapter Topological Visualisation Techniques for Volume Multifield Data, https://archive.org/details/oapen-20.500.12657-49310, fetched 2026-10-27 (unofficial)
