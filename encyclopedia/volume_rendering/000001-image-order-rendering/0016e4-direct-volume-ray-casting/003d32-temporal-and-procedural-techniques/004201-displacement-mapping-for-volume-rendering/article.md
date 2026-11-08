# Displacement Mapping for Volume Rendering
Also called: Discontinuous Displacement Mapping, Generalized Displacement Mapping, Inverse Displacement Mapping

## Overview
Displacement Mapping for Volume Rendering is a technique for modeling interactive deformations (cuts, peeling, deforming) on volumetric objects by applying unconventional, unorthogonal, and discontinuous displacements, avoiding mesh re-tessellation [S1, unofficial]. Unlike surface subdivision which requires costly re-meshing for discontinuities, this method uses GPU-based volumetric rendering without mesh structures [S1, unofficial]. It sets itself apart from standard procedural noise methods by allowing arbitrary, non-smooth geometric transformations of the volume domain itself, rather than just modulating density values [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A proxy scene geometry (bounding box) defining the spatial domain of the displaced object $O_0$; a displacement function $D$ (stored as a texture $D_k$); an original object representation $O$ (e.g., scalar volume dataset $O_j$); and a reference coordinate system $\Lambda$.
*   **Output:** A rendered image of the displaced volumetric object.

**Data Structures:**
*   **Displacement Texture ($D_k$):** Stores the discretization of the displacement function. Format is 8-bit or 16-bit point numbers, normalized in $[0, 1]$ [S1, unofficial].
*   **Volume Texture ($O_j$):** Stores the original scalar volume dataset in GPU memory [S1, unofficial].
*   **Proxy Geometry:** A bounding box object that defines the bounded spatial domain of the displaced object $O_0$ [S1, unofficial].

**Mathematical Formulation:**
Let $\Lambda$ be a common reference coordinate system shared by an object position function $P$ and a displacement function $D_B$. The forward displacement mapping is defined as [S1, unofficial]:
$$
P_0(\lambda) = P(\lambda) + D_B(\lambda)
$$
where $\lambda \in \Lambda$, $P$ maps reference to a point in $E^3$, and $D_B$ is the displacement vector function.

In this work, $\Lambda = E^3$ (3D Euclidean space) [S1, unofficial]. The rendering uses the inverse displacement mapping. Let $p$ be a point in the original object space and $p_0$ be a point in the displaced space. The relationship is rewritten as [S1, unofficial]:
$$
p = p_0 + D(p_0)
$$
where $D$ is the inverse displacement function $D_C$ (inverse of $D_B$) [S1, unofficial]. There is no constraint on the displacement values of $D$; it can be of any direction and magnitude [S1, unofficial].

**Algorithm Steps:**
1.  **Define Reference System:** Set the common reference coordinate system $\Lambda$ to 3D Euclidean space ($E^3$) to allow displacement in any direction and magnitude [S1, unofficial].
2.  **Setup Proxy Geometry:** Define a proxy scene geometry (a bounding box) to define the bounded spatial domain of the displaced object $O_0$ [S1, unofficial].
3.  **Render Slices:** Use texture-based volume rendering with view-aligned slices. Sample the proxy scene geometry with slices parallel to the view plane [S1, unofficial].
4.  **Inverse Mapping:** For each pixel in a slice, obtain the sampled point $p_0$ in the displaced space. Map this point back to the original object space $p$ using the inverse displacement map:
    $$
    p = p_0 + D(p_0)
    $$
    [S1, unofficial].
5.  **Attribute Lookup:** Evaluate the original object representation $O$ at the mapped point $p$ to retrieve luminance/density attributes. $O(p)$ can be an implicit surface function, level set, distance field, or color volume texture [S1, unofficial].
6.  **Normal Calculation:** Compute surface normals correctly to avoid lighting artifacts at breaking points in discontinuous displacement maps. Conventional normal estimation causes artifacts [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_displaced_volume(proxy_bounds, displacement_texture, volume_texture, view_matrix):
    """
    Renders a displaced volume using inverse displacement mapping.
    
    Args:
        proxy_bounds: Bounding box of the displaced object space O0.
        displacement_texture: 3D texture storing D(p0), normalized [0,1].
        volume_texture: 3D texture storing original object O(p).
        view_matrix: Camera view matrix for slice generation.
        
    Returns:
        Image buffer.
    """
    image = create_image_buffer()
    
    # Generate view-aligned slices through the proxy bounding box
    slices = generate_view_aligned_slices(proxy_bounds, view_matrix)
    
    for slice in slices:
        for pixel in slice.pixels:
            # 1. Sample point p0 in the displaced space (from proxy geometry/slice)
            p0 = pixel.world_position
            
            # 2. Sample displacement vector D(p0) from texture
            # Texture stores normalized values, must be scaled to actual displacement units
            # Assuming displacement texture is in same coordinate space as p0
            d_sample = displacement_texture.sample(p0)
            
            # Convert normalized texture value to actual displacement vector
            # [G] Implementation detail: The source states 8/16-bit normalized.
            # The actual vector D(p0) depends on the scale of the displacement function.
            # For a generic implementation, we assume the texture stores the vector components directly
            # or a scalar magnitude with a direction field. The source implies D is a vector function.
            # If D is stored as RGB vector:
            D_p0 = vec3(d_sample.r, d_sample.g, d_sample.b) * displacement_scale
            
            # 3. Compute original space point p using inverse mapping: p = p0 + D(p0)
            p = p0 + D_p0
            
            # 4. Check if p is within the valid domain of the original volume
            if not volume_texture.contains(p):
                continue
                
            # 5. Sample original object attributes at p
            density = volume_texture.sample(p)
            
            # 6. Compute correct normal at p to avoid artifacts at discontinuities
            # [G] The source notes conventional normal estimation fails.
            # A robust implementation would compute normals via finite differences 
            # of the displacement field or the original volume gradient in original space.
            normal = compute_correct_normal(p, displacement_texture, volume_texture)
            
            # 7. Shade pixel using density and normal
            color = shade_pixel(density, normal, light_source)
            
            image.set_pixel(pixel, color)
            
    return image
```

**Parameters:**
*   **Reference Space $\Lambda$:** $E^3$ [S1, unofficial].
*   **Displacement Texture Precision:** 8-bit or 16-bit [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Lighting Artifacts:** Conventional normal estimation causes noticeable lighting artifacts on surfaces displaced in varying directions and at breaking points in a discontinuous displacement map. Correct normal calculation is required [S1, unofficial].
*   **Discontinuity Handling:** The method supports discontinuous displacements (cuts, peeling) which are difficult for mesh-based approaches [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the number of slices and samples per slice. The inverse mapping $p = p_0 + D(p_0)$ is an $O(1)$ lookup per sample.
*   **Memory Complexity:** Requires storage for the proxy geometry, the displacement texture ($D_k$), and the original volume texture ($O_j$) [S1, unofficial].
*   **Strengths:** Allows interactive deformations (cuts, peeling) without re-meshing; supports unorthogonal and discontinuous displacements; leverages GPU texture hardware [S1, unofficial].
*   **Limitations:** Requires correct normal calculation to avoid artifacts; relies on a proxy geometry which may not perfectly bound complex deformations if not sized appropriately [S1, unofficial].
*   **When to use:** When modeling interactive volumetric deformations like cutting or peeling where mesh re-tessellation is prohibitive [S1, unofficial].
*   **When not to use:** When high-fidelity surface normals are critical and the displacement field is highly complex, requiring expensive normal computation [S1, unofficial].

## Background
Displacement mapping is commonly used for adding surface details to an object. Traditional methods involve micro-polygons, resulting in an explicit representation of the displaced surface, available in software like RenderMan and Maya, and hardware solutions [S1, unofficial]. Discontinuities in these methods are introduced with costly re-meshing of the object, which cannot be extended easily for volume graphics since no surface model is available [S1, unofficial]. In ray tracing, the conventional approach is to pre-compute an inverse displacement map [S1, unofficial]. This technique generalizes displacement mapping to volumetric objects by using inverse displacement maps in 3D vector space, allowing complex displacements to be applied to volumes without mesh structures [S1, unofficial].

## Sources
[S1] Discontinuous Displacement Mapping for Volume Graphics, https://doi.org/10.2312/vg/vg06/009-016, fetched 2026-10-29 (unofficial)
