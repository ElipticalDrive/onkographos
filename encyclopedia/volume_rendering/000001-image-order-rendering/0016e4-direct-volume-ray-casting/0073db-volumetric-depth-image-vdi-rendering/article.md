# Volumetric Depth Image (VDI) Rendering

Also called: Raycasting of Volumetric Depth Images, VDI raycasting

## Overview
Volumetric Depth Image (VDI) rendering is an image-order method that decouples expensive volume integration from interactive viewpoint changes by compositing pre-computed segments (supersegments) stored in a VDI, rather than integrating the volume rendering integral directly [S1]. It differs from standard direct volume ray casting by replacing per-pixel sample accumulation with geometric intersection tests against view-dependent supersegments, and from object-order methods by achieving frame rates an order of magnitude higher near the generation viewpoint while maintaining quality [S1].

## How it works
**Inputs and Outputs**
- **Input**: A Volumetric Depth Image (VDI) generated from a source viewpoint $V_0$, a new target viewpoint $V_N$ (defined by camera origin $\mathbf{o}_N$ and direction $\mathbf{d}_N$), and a transfer function $T$ (used during VDI generation, not rendering).
- **Output**: A 2D image where each pixel $(x, y)$ contains the accumulated color $\mathbf{C}_{out}$ and opacity $\alpha_{out}$.

**Data Structures**
The VDI consists of $N_L$ lists, one for each ray in the generation viewport.
- $N_L$: Number of lists, determined by the viewport resolution (number of rays). Typically $N_L \gg N_S$ [S1].
- $L_i$: The list of supersegments for ray $i$.
- $N_S$: The maximum number of supersegments allowed in each list $L_i$. This is a pre-set limit during VDI generation [S1].
- $S_j^i$: The $j$-th supersegment in list $L_i$. It stores:
    - $f(S_j^i)$: Distance from the near-plane to the front face of the supersegment.
    - $b(S_j^i)$: Distance from the near-plane to the back face of the supersegment.
    - $\mathbf{C}_j^i$: Accumulated color between $f(S_j^i)$ and $b(S_j^i)$.
    - $\alpha_j^i$: Accumulated opacity between $f(S_j^i)$ and $b(S_j^i)$.
- Fully transparent regions are excluded from supersegments [S1].

**Algorithm Steps**
1.  **Ray Definition**: For each pixel $(x, y)$ in the target image, define a ray $r(t) = \mathbf{o}_N + t\mathbf{d}_N$ [G].
2.  **NDC Conversion**: Convert the ray's start point $\mathbf{o}_N$ and a point far along the direction $\mathbf{o}_N + \mathbf{d}_N \cdot D_{max}$ (where $D_{max}$ is sufficiently large to cover the volume) from world space to the Normalized Device Coordinate (NDC) space of the generation viewpoint $V_0$. Let these be $\mathbf{p}_{start}^{NDC}$ and $\mathbf{p}_{end}^{NDC}$ [S1].
3.  **NDC Intersection**: Step through the list $L_i$ (corresponding to the pixel's ray in the generation viewport) in NDC space to determine intersection points $S$. In NDC space, the supersegments form simplified geometric shapes (rectangular frustums aligned with axes), allowing for efficient intersection calculation compared to irregular pyramidal frustums in world space [S1].
4.  **World Space Conversion**: Convert the intersection points $S$ determined in NDC space back to world space [S1].
5.  **Length Calculation**: Determine the length of the intersection segment in world space using the converted points. This length corresponds to the physical distance the ray travels through the supersegment [S1].
6.  **Compositing**: Composite the supersegments along the ray using the front-to-back volume rendering equation. For each intersected supersegment $S_j^i$ with accumulated color $\mathbf{C}_j^i$ and opacity $\alpha_j^i$, and assuming the ray traverses the full extent of the supersegment (or a fraction thereof if the ray exits the volume bounding box early), update the accumulator:
    $$
    \mathbf{C}_{acc} \leftarrow \mathbf{C}_{acc} + (1 - \alpha_{acc}) \cdot \mathbf{C}_j^i \cdot \alpha_j^i
    $$
    $$
    \alpha_{acc} \leftarrow \alpha_{acc} + (1 - \alpha_{acc}) \cdot \alpha_j^i
    $$
    Initialize $\mathbf{C}_{acc} = (0,0,0)$ and $\alpha_{acc} = 0$ .
7.  **Output**: Store $\mathbf{C}_{acc}$ and $\alpha_{acc}$ in the pixel buffer.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_vdi(vdi, camera_origin, camera_direction, image_width, image_height):
    """
    Renders an image using Volumetric Depth Image (VDI) raycasting.
    
    Args:
        vdi: A list of lists, where vdi[i] is the list L_i of supersegments for ray i.
             Each supersegment S is a tuple (f, b, C_acc, alpha_acc).
             f: distance from near-plane to front face.
             b: distance from near-plane to back face.
             C_acc: accumulated color (vec3).
             alpha_acc: accumulated opacity (float).
        camera_origin: vec3, origin of the new viewpoint V_N.
        camera_direction: vec3, viewing direction of V_N.
        image_width: int, width of the output image.
        image_height: int, height of the output image.
        
    Returns:
        image: 2D array of vec3, the rendered color image.
    """
    image = zeros((image_height, image_width, 3))
    
    # Pre-calculate NDC projection matrices for V_0 (generation viewpoint)
    # This assumes V_0's projection matrix P_0 is known from VDI generation.
    # NDC coordinates are in range [-1, 1] or [0, 1] depending on convention.
    # We assume standard OpenGL NDC [-1, 1].
    
    for y in range(image_height):
        for x in range(image_width):
            # 1. Define ray in world space
            # Convert pixel (x, y) to camera space direction
            # This requires the inverse projection matrix of V_N or standard perspective projection math
            # For simplicity, assume we have a function to get ray from pixel
            ray_o, ray_d = get_ray_from_pixel(x, y, camera_origin, camera_direction, image_width, image_height)
            
            # 2. Convert ray endpoints to NDC space of V_0
            # We need a point at t=0 (ray_o) and a point far away (ray_o + ray_d * large_value)
            large_value = 1000.0 # Sufficiently large to cover volume
            p_start_world = ray_o
            p_end_world = ray_o + ray_d * large_value
            
            # Project these points to NDC space of V_0
            # P_0 is the projection matrix of the generation viewpoint
            p_start_ndc = project_to_ndc(p_start_world, P_0)
            p_end_ndc = project_to_ndc(p_end_world, P_0)
            
            # 3. Determine which ray index i in VDI corresponds to this pixel
            # This mapping depends on how the VDI was generated relative to the viewport.
            # Typically, if the new viewpoint is close to V_0, the ray indices align.
            # If not, a more complex mapping is needed. The source implies a direct mapping
            # for the "generation viewpoint" context, but for arbitrary V_N, we assume
            # the VDI covers the entire volume and we map the ray to the closest VDI ray.
            # For this implementation, we assume a direct mapping for simplicity,
            # or that the VDI is dense enough that we can interpolate.
            # However, the source [S1] describes intersecting the list L_i.
            # Let's assume we find the closest ray index i in the VDI.
            i = find_closest_vdi_ray_index(p_start_ndc, vdi)
            
            L_i = vdi[i]
            
            # 4. Step through L_i in NDC space to find intersections
            # In NDC space, the supersegments are axis-aligned frustums.
            # We check if the ray segment (p_start_ndc to p_end_ndc) intersects each supersegment.
            intersections = []
            for S in L_i:
                f_ndc = S.f # Distance in NDC? No, f and b are distances from near-plane in world space during generation.
                # Wait, the source says: "Each ray i generates a list L_i of supersegments S_j^i that store the distances 
                # between the near-plane and their front and back faces, f(S_j^i) and b(S_j^i)".
                # And: "Compute ray-segment intersections in Normalized Device Coordinate (NDC) space... 
                # as segments form pyramidal frustums in world space but have simplified intersections in NDC."
                # This implies that during rendering, we convert the ray to NDC, and the supersegments are also represented 
                # in NDC? Or we convert the ray to NDC and intersect with the NDC representation of the supersegments.
                # The source says: "Once S_j^i intersection points are determined in NDC space, they are converted back to world space".
                # So we need the NDC bounds of each supersegment.
                # Let's assume the VDI stores NDC bounds for each supersegment, or we can compute them from f and b.
                # Since f and b are world-space distances from the near plane, and the near plane is at z=-1 (or 0) in NDC,
                # we can map f and b to NDC z-coordinates.
                # However, the source explicitly mentions converting the ray to NDC and intersecting there.
                # Let's assume the VDI stores the NDC bounds for each supersegment for efficiency.
                # If not, we must compute them. Given the source's emphasis on NDC intersection, we assume NDC bounds are available.
                
                # For the sake of a complete implementation, let's assume the VDI stores NDC bounds.
                # If the VDI only stores f and b (world space), we need to project the front and back planes of the supersegment to NDC.
                # This is complex. The source says "simplified way of intersecting rays with segments by computing them in NDC space".
                # This implies the supersegments are treated as simple boxes in NDC.
                
                # Let's assume we have a function to get NDC bounds of a supersegment.
                ndc_bounds = get_ndc_bounds(S, P_0)
                
                # Intersect ray segment (p_start_ndc, p_end_ndc) with ndc_bounds
                hit = intersect_ray_ndc_segment(p_start_ndc, p_end_ndc, ndc_bounds)
                if hit is not None:
                    intersections.append((S, hit))
            
            # 5. Convert intersection points back to world space and calculate lengths
            # 6. Composite
            C_acc = vec3(0, 0, 0)
            alpha_acc = 0.0
            
            # Sort intersections by distance from camera
            intersections.sort(key=lambda x: x[1].t)
            
            for S, hit in intersections:
                # Convert hit point from NDC to world space
                hit_world = unproject_ndc(hit.point, P_0)
                
                # Calculate the length of the intersection in world space
                # The hit object should contain the entry and exit points in NDC
                # Convert both to world space
                entry_world = unproject_ndc(hit.entry_ndc, P_0)
                exit_world = unproject_ndc(hit.exit_ndc, P_0)
                
                length = distance(entry_world, exit_world)
                
                # The supersegment's accumulated color and opacity are for the entire segment.
                # If the ray only traverses a part of the segment, we need to scale the opacity.
                # However, the source says "supersegments storing distances and accumulated color/opacity".
                # And "fully transparent regions are excluded".
                # The compositing equation uses alpha_i. If the ray doesn't traverse the full segment,
                # we need to adjust alpha_i.
                # The source doesn't explicitly detail partial traversal compositing.
                # Assuming the ray traverses the full segment for simplicity, or that the VDI segments are small enough.
                # If partial traversal is needed, alpha_i should be scaled by (length / segment_length).
                # But the VDI stores f and b, which are the full extent.
                # Let's assume the ray traverses the full segment for now, as the source focuses on intersection simplification.
                
                # Update accumulator
                # C_acc = C_acc + (1 - alpha_acc) * C_j^i * alpha_j^i
                # alpha_acc = alpha_acc + (1 - alpha_acc) * alpha_j^i
                
                # If partial traversal, we need to compute the effective alpha for the traversed length.
                # This requires knowing the extinction coefficient or assuming uniform opacity.
                # The source does not provide this detail. We assume full traversal for the basic algorithm.
                
                C_acc = C_acc + (1 - alpha_acc) * S.C_acc * S.alpha_acc
                alpha_acc = alpha_acc + (1 - alpha_acc) * S.alpha_acc
                
            image[y, x] = C_acc
            
    return image

def project_to_ndc(point_world, P):
    """Projects a world space point to NDC space using projection matrix P."""
    point_cam = P * vec4(point_world, 1.0)
    if point_cam.w == 0:
        return None
    ndc = point_cam.xyz / point_cam.w
    return ndc

def unproject_ndc(ndc_point, P):
    """Unprojects an NDC point to world space using inverse projection matrix P_inv."""
    P_inv = inverse(P)
    point_cam = vec4(ndc_point, 1.0)
    point_world_hom = P_inv * point_cam
    if point_world_hom.w == 0:
        return None
    return point_world_hom.xyz / point_world_hom.w

def get_ndc_bounds(supersegment, P):
    """Returns the NDC bounds (min_x, max_x, min_y, max_y, min_z, max_z) of a supersegment."""
    # This requires knowing the geometry of the supersegment in world space.
    # The source says supersegments are irregular pyramidal frustums in world space.
    # In NDC, they become axis-aligned boxes.
    # We need to project the 8 corners of the frustum to NDC and find the min/max.
    # This is computationally expensive. The source implies this is done efficiently.
    # For the purpose of this article, we assume this function exists and is efficient.
    pass

def intersect_ray_ndc_segment(ray_start_ndc, ray_end_ndc, ndc_bounds):
    """Intersects a ray segment with an axis-aligned box in NDC space."""
    # Standard slab method for ray-box intersection
    pass

def find_closest_vdi_ray_index(ndc_point, vdi):
    """Finds the index i in the VDI closest to the given NDC point."""
    # This depends on the VDI generation. If the VDI is generated for a specific viewport,
    # the NDC coordinates map directly to pixel indices.
    # If the new viewpoint is different, the mapping is not direct.
    # The source mentions "decoupling viewpoint changes", implying the VDI is view-dependent.
    # For a general implementation, this step is complex.
    # We assume a simple mapping for the sake of the algorithm description.
    pass
```

**Parameters**
- $N_S$: Maximum number of supersegments per list. Pre-set during VDI generation [S1].
- $N_L$: Number of lists, equal to the viewport resolution during VDI generation [S1].

**Edge Cases and Pitfalls**
- **Irregular Geometry**: VDI segments are irregular pyramidal frustums in world space due to perspective projection, making direct intersection calculations complex. This is mitigated by performing intersections in NDC space [S1].
- **Viewpoint Dependency**: The VDI is view-dependent. Rendering quality degrades as the new viewpoint moves away from the generation viewpoint $V_0$ [S1].
- **Memory Access**: The method minimizes memory accesses by exploiting spatial homogeneity in the data [S1].

## Operating characteristics
- **Time Complexity**: The rendering time is dominated by the intersection tests in NDC space and the compositing of supersegments. It is generally faster than standard ray casting because it avoids per-sample volume texture lookups and transfer function evaluations [S1].
- **Memory Complexity**: The VDI size is proportional to $N_L \times N_S$. Since $N_L \gg N_S$, the memory footprint is significantly smaller than storing the full volume for high-resolution rendering [S1].
- **Performance**: Achieves frame rates an order of magnitude higher than previous object-order rendering algorithms near the generation viewpoint, while maintaining quality comparable to ground-truth volume rendering [S1].
- **Strengths**: Efficient for remote visualization, interactive frame rates for high-resolution displays, reduced calculations and memory accesses compared to previous raycasting approaches [S1].
- **Limitations**: Quality degrades with viewpoint distance from $V_0$. Requires pre-computation and streaming of VDIs [S1].

## Background
VDI rendering was introduced by Frey et al. [10] as a compact view-dependent volume representation. The method was further optimized for efficient raycasting by Brady et al. [5] and Lochmann et al. [21]. The specific algorithm described here, which simplifies intersections in NDC space, was proposed in 2023 [S1]. It is used in remote visualization settings to enable interactive visualization of large volumes at high frame rates [S1].

## Sources
[S1] Efficient Raycasting of Volumetric Depth Images for Remote Visualization of Large Volumes at High Frame Rates, https://doi.org/10.1109/pacificvis56936.2023.00014, fetched 2026-10-24
