# Compositional Rendering
Also called: GO-NeRF Compositional Rendering, GO-NeRF Rendering

## Overview
Compositional Rendering is a pipeline that decouples the rendering of a pre-existing 3D scene from the generation of new objects, merging them via a 3D-aware opacity map to handle occlusions effectively [S1]. It differs from standard direct volume ray casting by pre-rendering the static background into RGB-D image sequences and compositing newly generated object volumes in image space, rather than integrating all geometry into a single unified volume traversal [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A pre-trained scene NeRF $F(\theta_s)$, a pre-trained object NeRF $F(\theta_o)$, a set of predefined camera viewpoints $\{C_0, \dots, C_n\}$, and a 3D bounding box defining the object's spatial extent.
*   **Outputs:** A final composited RGB image $I_v$ for each viewpoint $v$.

**Data Structures:**
*   **Pre-rendered Scene:** A sequence of RGB-D images $\{(S_0, D_0), \dots, (S_n, D_n)\}$, where $S_v$ is the RGB image and $D_v$ is the depth map of the background scene at viewpoint $v$ [S1].
*   **Object Volume:** Defined by the object NeRF $F(\theta_o)$, providing volume density $\delta$ and RGB color $c$ at query points [S1].
*   **Bounding Box:** A 3D axis-aligned box defining the region where the object exists [S1].

**Numbered Steps:**

1.  **Pre-render the Scene:** Generate RGB-D images for the established scene NeRF $F(\theta_s)$ from the predefined camera viewpoints $\{C_0, \dots, C_n\}$ [S1]. These are stored as $\{(S_0, D_0), \dots, (S_n, D_n)\}$ to avoid repeated volume rendering during object optimization [S1].
2.  **Ray Casting for Object:** For a specific viewpoint $v$, cast rays $r_i$ from the camera center. Sample query points $q_k$ along the ray within the 3D bounding box of the object [S1].
3.  **Compute Transmittance:** Calculate the transmittance $\tau_k$ at sample point $k$ using the volume density $\delta_t$ and distance $\Delta_t$ between adjacent points [S1]:
    $$ \tau_k = \exp\left(-\sum_{t=1}^{k-1} \delta_t \Delta_t\right) $$
    where $\delta_t$ is the volume density and $\Delta_t$ is the distance between adjacent query points [S1].
4.  **Render Object RGB:** Compute the RGB value $G_v(i)$ for ray $r_i$ by accumulating the color contribution of samples inside the box [S1]:
    $$ G_v(i) = \sum_{k=1}^{K} \tau_k (1 - \exp(-\delta_k \Delta_k)) c_k $$
    where $c_k$ is the RGB value at query point $q_k$ [S1].
5.  **Render Object Opacity with Occlusion:** Compute the opacity map $O_v(i)$ for ray $r_i$. The opacity is the accumulated alpha of the object, but it is clamped to 0 if the ray does not intersect the bounding box or if the object is occluded by the pre-rendered scene foreground [S1]:
    $$ O_v(i) = \begin{cases} \sum_{k=1}^{K} \tau_k (1 - \exp(-\delta_k \Delta_k)), & \text{if } r_i \in \text{box} \text{ and } D_{\text{box}}(i) \le D_v(i) \\ 0, & \text{otherwise} \end{cases} $$
    Here, $D_{\text{box}}(i)$ is the depth of the object's bounding box intersection (or the nearest object sample), and $D_v(i)$ is the depth of the pre-rendered scene [S1].
6.  **Composite Final Image:** Blend the rendered object $G_v$ and the pre-rendered scene $S_v$ using the opacity map $O_v$ [S1]:
    $$ I_v = G_v \cdot O_v + S_v \cdot (1 - O_v) $$

**Parameters:**
*   **Number of samples ($K$):** Not given in the sources.
*   **Bounding Box:** Defined by the user or generation pipeline [S1].

**Edge Cases and Pitfalls:**
*   **No Intersection:** If the ray $r_i$ does not intersect the 3D bounding box, the opacity $O_v(i)$ is set to 0, ensuring the original scene content $S_v$ is preserved [S1].
*   **Occlusion:** If the generated object is behind the scene foreground ($D_{\text{box}}(i) > D_v(i)$), the opacity $O_v(i)$ is set to 0 to prevent the object from appearing in front of the scene [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compositional_rendering(scene_nerf, object_nerf, camera_viewpoint, bounding_box, pre_rendered_scene_data):
    """
    Renders a composited image by blending a pre-rendered scene with a generated object.
    
    Args:
        scene_nerf: The pre-trained NeRF for the background scene.
        object_nerf: The NeRF for the object to be composited.
        camera_viewpoint: The camera parameters for the current view.
        bounding_box: The 3D bounding box for the object.
        pre_rendered_scene_data: Dictionary mapping camera indices to (S_v, D_v) tuples.
        
    Returns:
        I_v: The final composited RGB image.
    """
    # 1. Retrieve pre-rendered scene data for this viewpoint
    # In practice, this is looked up from the pre-computed sequence {(S0, D0), ...}
    S_v, D_v = pre_rendered_scene_data[camera_viewpoint.index]
    
    # 2. Cast rays and sample object volume
    rays = generate_rays(camera_viewpoint)
    K = number_of_samples_per_ray # Not given in sources, assumed fixed or adaptive
    G_v = zeros_like(S_v) # Object RGB
    O_v = zeros_like(S_v) # Object Opacity
    
    for i, ray in enumerate(rays):
        # Sample points along the ray within the bounding box
        query_points = sample_along_ray(ray, bounding_box, K)
        
        # Get density and color from object NeRF
        densities = []
        colors = []
        for q in query_points:
            delta, c = object_nerf.query(q)
            densities.append(delta)
            colors.append(c)
            
        # Calculate distances between adjacent points
        deltas = []
        for j in range(len(query_points) - 1):
            dist = distance(query_points[j], query_points[j+1])
            deltas.append(dist)
        # Handle last segment or assume uniform spacing if not specified
        # [G] Assuming uniform spacing or last delta is zero if not defined
        if len(deltas) < len(densities):
            deltas.append(0.0) 
            
        # 3. Compute Transmittance and Accumulate
        tau = 1.0
        acc_color = 0.0
        acc_opacity = 0.0
        
        for k in range(len(densities)):
            delta_k = densities[k]
            c_k = colors[k]
            delta_dist = deltas[k]
            
            # Transmittance to current point
            # tau_k = exp(-sum_{t=1}^{k-1} delta_t * dist_t)
            # We maintain tau as the transmittance *before* the current sample
            contribution = tau * (1 - exp(-delta_k * delta_dist))
            
            acc_color += contribution * c_k
            acc_opacity += contribution
            
            # Update transmittance for next sample
            tau *= exp(-delta_k * delta_dist)
            
        # 4. Determine Occlusion
        # Check if ray intersects bounding box
        intersects_box = ray_intersects_box(ray, bounding_box)
        
        # Check depth occlusion
        # D_box(i) is the depth of the object. 
        # [G] The source implies D_box(i) is derived from the object's depth or box intersection.
        # If the object NeRF provides depth, use it. Otherwise, use the box intersection depth.
        if intersects_box:
            # [G] Assuming object_nerf can provide depth or we use the first/last sample depth
            # The source says D_box(i) > D_v(i) implies occlusion.
            # We need the depth of the object along this ray.
            # Let's assume we can get the depth of the nearest object sample or box entry.
            # For simplicity, if we have query points, we might use the depth of the first non-zero density point.
            # However, the source formula uses D_box(i). 
            # [G] If D_box is not explicitly provided by the NeRF, it is the depth of the bounding box intersection.
            depth_box = get_depth_at_ray_intersection(ray, bounding_box) 
            is_occluded = depth_box > D_v[i]
        else:
            is_occluded = True # No intersection means no object
            
        # 5. Set Opacity based on occlusion
        if intersects_box and not is_occluded:
            O_v[i] = acc_opacity
            G_v[i] = acc_color
        else:
            O_v[i] = 0.0
            G_v[i] = 0.0 # Color doesn't matter if opacity is 0
            
    # 6. Composite
    # I_v = G_v * O_v + S_v * (1 - O_v)
    I_v = G_v * O_v + S_v * (1 - O_v)
    
    return I_v
```

## Operating characteristics
*   **Time Complexity:** The method reduces computational load during object optimization by pre-rendering the scene [S1]. The cost is dominated by the object NeRF evaluation and the final image-space composition, which is $O(N)$ per pixel where $N$ is the number of samples [S1].
*   **Memory Complexity:** Requires storing the pre-rendered RGB-D image sequences for all viewpoints $\{(S_0, D_0), \dots, (S_n, D_n)\}$ [S1].
*   **Strengths:** Decouples object and scene rendering, enhancing flexibility and allowing for plug-and-play integration with various scene pre-training methods [S1]. Handles occlusions effectively by comparing 3D depths [S1].
*   **Limitations:** Relies on a predefined set of camera viewpoints for pre-rendering, which may limit free-viewpoint navigation outside these views [S1].

## Background
Compositional Rendering was introduced in the context of GO-NeRF for generating objects in Neural Radiance Fields for virtual reality content creation [S1]. It addresses the challenge of seamlessly integrating newly generated 3D objects into existing 3D scenes while maintaining correct occlusions and visual consistency [S1].

## Sources
[S1] GO-NeRF: Generating Objects in Neural Radiance Fields for Virtual Reality Content Creation, https://doi.org/10.1109/tvcg.2025.3549558, fetched 2026-10-29
