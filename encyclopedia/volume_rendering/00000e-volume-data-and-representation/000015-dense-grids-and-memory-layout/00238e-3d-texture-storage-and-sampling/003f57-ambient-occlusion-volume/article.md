# Ambient Occlusion Volume

Also called: 3D Texture AO, AOV rendering

## Overview
Ambient Occlusion Volume (AOV) is a method for storing precomputed near-field ambient occlusion factors directly in a 3D texture, optionally alongside bent normals, to avoid runtime reconstruction computation [S1, unofficial]. It approximates the near-field ambient occlusion problem by computing an analytic shadow-volume-like enclosure around each polygon and applying a tunable occlusion function within that region [S2, unofficial]. Unlike Kontkanen and Laine’s approach, which requires runtime reconstruction of occlusion factors, AOV eliminates this cost by baking the factors into the texture [S1, unofficial]. It differs from disk-based methods (e.g., Bunnell, Hoberock/Jia) and Shopf et al.’s bounding cube method by using tight dynamic bounding volumes and partial coverage to avoid over-estimating occlusion and double-occlusion artifacts [S2, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A dynamic triangle mesh representing the scene geometry.
- **Output:** A 3D texture (the Ambient Occlusion Volume) storing ambient occlusion factors. Optionally, bent normals are stored alongside the occlusion factors.

**Data Structures:**
- **3D Texture:** Stores the ambient occlusion factor per voxel. The layout is a dense 3D grid. If bent normals are included, they are stored as additional channels (e.g., RGB) alongside the occlusion factor (e.g., R or RGBA) [S1, unofficial]. The exact bit-depth and channel packing are not specified in the sources [G: Typically 16-bit or 32-bit float for precision, but not given].

**Parameters:**
- **$\delta$ (delta):** The effective occlusion distance limit. It defines the radius of the imaginary sphere around a point $x$ for ambient illumination calculation. Typical value: 1m for human-scale scenes [S2, unofficial].

**Algorithm Steps:**
1. **Tessellation:** Curved surfaces must be tessellated into triangle meshes, as the algorithm operates on polygons [S2, unofficial].
2. **Polygon Processing:** For each polygon $P$ with vertices $\{\vec{p}_0, \dots, \vec{p}_{k-1}\}$, compute its contribution to the ambient occlusion volume.
3. **Analytic Occlusion Calculation:** For an infinitesimal patch $X$ at the origin with normal $\hat{n}$, and a polygon $P$ lying in the positive half-plane ($\vec{p} \cdot \hat{n} \ge 0$), the occlusion $\text{AO}_P(\hat{n})$ is calculated using the form factor for diffuse radiative transfer [S2, unofficial]:
   $$ \text{AO}_P(\hat{n}) = \frac{1}{2\pi} \sum_{i=0}^{k-1} \cos^{-1} \left( \frac{\vec{p}_i \cdot \vec{p}_j}{\|\vec{p}_i\| \|\vec{p}_j\|} \right) \hat{n} \cdot \frac{\vec{p}_i \times \vec{p}_j}{\|\vec{p}_i \times \vec{p}_j\|} $$
   where $j = (i+1) \mod k$ [S2, unofficial].
   - Implementation note: This requires 1 arccosine, 15 multiply-add, and 2 reciprocal square root operations per edge [S2, unofficial].
4. **Fractional Obscurance:** To avoid hard cutoffs at distance $\delta$, binary occlusion is replaced with fractional obscurance. The visibility function $V(\hat{\omega}_i)$ is 1 if there is an unoccluded line of sight between $x$ and $x+\hat{\omega}_i\delta$, and 0 otherwise. The term $(1 - V(x,y))$ is replaced by $(1 - V(x,y))g(x,y)$, where $g$ is a smooth, monotonic falloff function such that $g(x,y) = 1$ at $x=y$ and $g(x,y) = 0$ for $\|x-y\| \ge \delta$ [S2, unofficial].
5. **Volume Rendering/Storage:** The contributions from all polygons are accumulated into the 3D texture. The algorithm is viewer-independent [S2, unofficial].
6. **Optional Upsampling:** Since the analytic solution is noise-free, upsampling (e.g., joint-bilateral) is optional and introduces error if applied unnecessarily [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_polygon_ao(polygon_vertices, normal_n, delta):
    """
    Computes the analytic ambient occlusion contribution of a polygon
    to an infinitesimal patch at the origin with normal n.
    """
    ao_sum = 0.0
    k = len(polygon_vertices)
    for i in range(k):
        p_i = polygon_vertices[i]
        p_j = polygon_vertices[(i + 1) % k]
        
        # Compute dot product and norms
        dot_product = dot(p_i, p_j)
        norm_p_i = length(p_i)
        norm_p_j = length(p_j)
        
        # Avoid division by zero if vertex is at origin
        if norm_p_i < 1e-9 or norm_p_j < 1e-9:
            continue
            
        # Compute cross product for the normal component
        cross_prod = cross(p_i, p_j)
        norm_cross = length(cross_prod)
        
        if norm_cross < 1e-9:
            continue
            
        # Compute the angle term
        # Clamp argument to [-1, 1] for numerical stability
        arg = dot_product / (norm_p_i * norm_p_j)
        arg = max(-1.0, min(1.0, arg))
        angle = acos(arg)
        
        # Compute the normal component
        normal_component = dot(normal_n, cross_prod / norm_cross)
        
        # Accumulate
        ao_sum += angle * normal_component
        
    # Final AO contribution
    ao = ao_sum / (2.0 * pi)
    return ao

def apply_falloff(distance, delta):
    """
    Applies the fractional obscurance falloff function g.
    g is smooth, monotonic, 1 at distance 0, and 0 at distance >= delta.
    """
    if distance >= delta:
        return 0.0
    # Example smooth falloff (not specified in sources, [G])
    # Using a simple quadratic falloff for demonstration
    t = distance / delta
    return 1.0 - t * t

def render_ao_volume(mesh, delta):
    """
    Renders the ambient occlusion volume for a given mesh.
    """
    # Initialize 3D texture with zeros
    ao_volume = initialize_3d_texture(mesh.bounding_box, resolution)
    
    for polygon in mesh.polygons:
        # For each voxel in the volume, compute contribution
        # Note: The paper describes the algorithm as operating on the mesh
        # and storing results in the volume. The exact rasterization or
        # voxelization step is not detailed in the provided notes.
        # [G: Typically, this involves rasterizing the polygon's influence
        # into the volume or ray-marching the volume against the polygon.
        # Since the source says "computes the analog of a shadow volume...
        # and applies a tunable occlusion function", we assume a rasterization
        # or volume integration step.]
        
        # Placeholder for the volume accumulation step
        # The analytic AO_P(n) is computed for the patch normal.
        # In a volume context, this would be integrated over the volume.
        pass
        
    return ao_volume
```

**Edge Cases and Pitfalls:**
- **Over-counting:** Occlusion is over-counted where thin objects are stacked [S2, unofficial].
- **Curved Surfaces:** Curved surfaces must be tessellated into meshes; the algorithm does not handle NURBS or implicit surfaces directly [S2, unofficial].
- **Hard Cutoffs:** Without the fractional obscurance falloff, hard cutoffs at $\delta$ would reveal transitions between occlusion methods [S2, unofficial].

## Subtopics

### [3D Texture Proxy Geometry](870-3d-texture-proxy-geometry.md)
Uses proxy geometry to approximate volume data. Choose when memory is limited and geometry is simple.

### [Volume Texture Creation](1857-volume-texture-creation.md)
General methods for creating volume textures. Choose for general volume data preparation.

### [Texture Uniting (Multi-Channel Packing)](2470-texture-uniting-multi-channel-packing.md)
Packs multiple data channels into a single texture. Choose when memory bandwidth is a bottleneck.

### [PixelFlow Volume Rendering](3443-pixelflow-volume-rendering.md)
A specific rendering pipeline. Choose for real-time volume rendering with specific hardware optimizations.

### [Texture Array Sampling with Alpha Discard](3993-texture-array-sampling-with-alpha-discard.md)
Uses texture arrays for discrete volumes. Choose for discrete, non-volumetric data.

### [Hardware Volume Rendering Pipeline with Pixel Textures](5641-hardware-volume-rendering-pipeline-with-pixel-textures.md)
Leverages hardware pixel textures. Choose for older hardware or specific pipeline constraints.

### [Sparse Volume Texture (SVT)](7011-sparse-volume-texture-svt.md)
Uses sparse representations. Choose for large, sparse volumes.

### [Memory Organization and Layout](7154-memory-organization-and-layout.md)
General memory layout principles. Choose for optimizing memory access patterns.

### [3D Volume Atlas Packing](7158-3d-volume-atlas-packing.md)
Packs multiple volumes into an atlas. Choose for managing multiple small volumes.

## Operating characteristics
- **Time Complexity:** Performance is largely independent of geometric complexity and is dominated by fill rate [S2, unofficial].
- **Memory Complexity:** Depends on the resolution of the 3D texture. Not given in the sources.
- **Measured Performance:** On the Sponza scene at 1280×720, a 60-ms AOV render has comparable quality to a 5-minute ray-traced result [S2, unofficial]. On a 1.4M-triangle scene at 1280×720, AOV results have quality comparable to ray tracing 1200 visibility samples per pixel but run in real time, achieving a 4000x speedup [S2, unofficial].
- **Strengths:** Viewer-independent, no noise or aliasing (beyond rasterization), real-time performance, analytic solution [S2, unofficial].
- **Limitations:** Over-counts occlusion for stacked thin objects, requires tessellation of curved surfaces [S2, unofficial].
- **When to Use:** When real-time performance is required and the scene consists of dynamic triangle meshes.
- **When Not to Use:** When high accuracy is required for thin, stacked objects or when curved surfaces cannot be tessellated.

## Background
Ambient Occlusion Volumes were introduced by M. McGuire from NVIDIA and Williams College in 2010 [S2, unofficial]. The method combines known pieces in a new way to achieve improved quality over fast methods and improved performance compared to accurate methods [S2, unofficial]. It is conceptually an extension of Kontkanen and Laine’s occlusion fields for objects to individual, dynamic triangles in a mesh, using techniques borrowed from shadow volumes [S2, unofficial]. The analytic solution for polygon occlusion was first introduced to graphics by Baum et al. in the context of the radiosity algorithm [S2, unofficial].

## Sources
[S1] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
[S2] Ambient Occlusion Volumes, https://research.nvidia.com/sites/default/files/pubs/2010-06_Ambient-Occlusion-Volumes/McGuire10AOV.pdf, fetched 2026-10-28 (unofficial)
