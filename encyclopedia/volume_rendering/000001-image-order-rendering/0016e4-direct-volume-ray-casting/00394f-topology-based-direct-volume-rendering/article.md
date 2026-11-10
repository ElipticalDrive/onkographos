# Topology-Based Direct Volume Rendering

Also called: Topology in direct volume rendering

## Overview
Topology-Based Direct Volume Rendering applies topological segmentation to scalar fields to guide direct volume rendering attributes (opacity, color), rather than relying solely on indirect mesh visualization [S1, unofficial]. It solves the problem of visually distinguishing connected topological structures (such as extrema and saddles) by using topological boundaries to modulate rendering properties, allowing inner features to remain visible while outer structures are attenuated [S1, unofficial]. It sets itself apart from standard direct volume ray casting by decoupling the classification logic from local scalar values and transfer functions, instead using global topological relationships (e.g., contour tree branches) to determine opacity levels [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A scalar volume dataset $V$, a topological segmentation structure (e.g., contour tree or volume skeleton tree) derived from $V$, and a camera view.
- Output: A 2D image with colors and opacities modulated by topological attributes.

**Data Structures:**
- **Scalar Volume:** A 3D grid of scalar values.
- **Topological Segmentation:** A data structure representing the topology of the scalar field. Common forms include:
    - **Contour Tree:** A graph where nodes represent critical points (maxima, minima, saddles) and edges represent intervals of constant topology.
    - **Volume Skeleton Tree:** A simplified representation of the connectivity of local extrema.
- **Topological Attributes:** Metadata associated with regions or boundaries in the segmentation, such as:
    - Persistence (difference in scalar value between a critical point and its saddle).
    - Occlusion level (depth or nesting within the topological hierarchy).
    - Number of equal-valued contours.

**Steps:**
1. **Topological Segmentation:** Compute the topological structure (e.g., contour tree) for the scalar field. Identify boundaries between topological objects (e.g., regions connected by specific contour tree branches) [S1, unofficial].
2. **Attribute Assignment:** Assign rendering attributes to samples or regions based on their topological properties. For example:
    - Assign opacity based on the number of equal-valued contours or occlusion level [S1, unofficial].
    - Assign lower opacity to outermost objects (those with fewer enclosing contours or higher in the hierarchy) to prevent them from obscuring inner features [S1, unofficial].
    - Assign color based on topological region identity (e.g., different colors for different extrema branches) [S1, unofficial].
3. **Direct Volume Rendering:** Perform standard direct volume ray casting [19-direct-volume-ray-casting.md].
    - For each sample along the ray, retrieve the scalar value $V(\mathbf{p}_i)$.
    - Determine the topological attribute for the region containing $\mathbf{p}_i$.
    - Map the topological attribute to opacity $\alpha_i$ and color $\mathbf{c}_i$. This mapping replaces or augments the standard transfer function $T(V) \rightarrow (\mathbf{c}, \alpha)$ [S1, unofficial].
    - Composite the samples using the front-to-back equation [19-direct-volume-ray-casting.md].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def topology_based_dvr(volume, contour_tree, camera):
    # 1. Precompute topological segmentation
    # contour_tree contains nodes (critical points) and edges (intervals)
    # Each voxel or region is labeled with a topological ID or attribute
    topological_labels = compute_contour_tree_labels(volume, contour_tree)
    
    # 2. For each pixel, cast a ray
    for pixel in camera.image_plane:
        ray = camera.cast_ray(pixel)
        samples = ray.sample(volume.bounds, step_size=volume.resolution)
        
        acc_color = (0, 0, 0)
        acc_alpha = 0.0
        
        for sample in samples:
            # Get scalar value
            v_val = volume.interpolate(sample.position)
            
            # Get topological attribute for this sample
            # This could be a label, persistence, or occlusion level
            topo_attr = topological_labels.get_attribute(sample.position)
            
            # Map topological attribute to color and opacity
            # This is the key difference from standard DVR
            color, opacity = map_topological_attribute_to_rendering(topo_attr)
            
            # Standard front-to-back compositing
            # c_out = c_acc + (1 - a_acc) * c_sample * a_sample
            # a_out = a_acc + (1 - a_acc) * a_sample
            weight = (1.0 - acc_alpha) * opacity
            acc_color = acc_color + weight * color
            acc_alpha = acc_alpha + weight
            
            if acc_alpha > 1.0:
                break # Early ray termination
                
        pixel.color = acc_color
```

**Parameters:**
- **Segmentation Algorithm:** Contour tree or volume skeleton tree [S1, unofficial].
- **Attribute Mapping Function:** Defines how topological metrics (e.g., occlusion, persistence) map to opacity/color. Takeshima et al. use occlusion to assign lower opacity to outer objects [S1, unofficial].
- **Step Size:** Standard ray marching step size [19-direct-volume-ray-casting.md].

**Edge Cases:**
- **Degenerate Topology:** Flat regions or constant scalar values may not have well-defined critical points, requiring special handling in the contour tree construction [S1, unofficial].
- **Precision:** Topological simplification (e.g., pruning small branches) can affect the rendering attributes if not handled consistently [S1, unofficial].

## Subtopics

### [Hardware-Accelerated Topological Volume Rendering](11664-hardware-accelerated-topological-volume-rendering.md)
Applies topological segmentation to guide direct volume rendering attributes by dynamically rendering "tubes" between saddles and extrema within the fragment shader, enabling real-time manipulation of topological simplifications like branch pruning.

## Operating characteristics
- **Time Complexity:** Dominated by the topological segmentation step (e.g., contour tree construction, which is $O(N \log N)$ or similar depending on the algorithm) and the standard ray casting step [S1, unofficial]. The per-sample cost is similar to standard DVR, plus the cost of looking up topological attributes.
- **Memory Complexity:** Requires storage for the scalar volume and the topological structure (contour tree/skeleton), which can be significant for large datasets [S1, unofficial].
- **Strengths:** Improves visual clarity by emphasizing inner structures and reducing clutter from outer structures; provides a semantically meaningful classification based on global topology rather than local scalar values [S1, unofficial].
- **Limitations:** Requires pre-computation of topological structures, which can be expensive; the quality of the rendering depends on the accuracy of the topological segmentation; may not be suitable for real-time applications without hardware acceleration or simplification [S1, unofficial].
- **When to Use:** When the goal is to visualize the hierarchical structure of a scalar field and distinguish between connected components based on their topological relationship [S1, unofficial].
- **When Not to Use:** When real-time performance is critical and topological pre-computation is too slow; when the scalar field does not have a clear topological structure (e.g., noisy data) [S1, unofficial].

## Background
Topology-based direct volume rendering was introduced as an alternative to mesh-based topological visualization, allowing topological information to guide the rendering attributes directly [S1, unofficial]. Takeshima et al. [44] (cited in [S1, unofficial]) were among the early proponents, using attributes like the number of equal-valued contours and occlusion to assign opacity levels. The approach has been further developed with hardware acceleration to enable interactive exploration of topological simplifications [S1, unofficial].

## Sources
[S1] Chapter Topological Visualisation Techniques for Volume Multifield Data, https://archive.org/details/oapen-20.500.12657-49310, fetched 2026-10-27 (unofficial)
