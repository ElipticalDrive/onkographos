# Compositional Rendering

Also called: GO-NeRF Compositional Rendering, GO-NeRF Rendering

## Overview
Compositional Rendering is a method for integrating newly generated 3D objects into an existing Neural Radiance Field (NeRF) scene by decoupling the rendering of the object from the background scene [S1]. It solves the problem of occlusion and seamless blending in virtual reality content creation by using a 3D-aware opacity map to merge the rendered object with pre-rendered scene buffers [S1]. Unlike standard volume rendering which integrates a single volume, this method composites two distinct radiance fields (scene and object) in image space, using depth comparisons to handle 3D occlusions correctly [S1].

## How it works
The method operates in two phases: pre-rendering the static scene and rendering the dynamic object, followed by image-space composition.

**Inputs and Outputs:**
*   **Inputs:**
    *   Pre-trained scene NeRF $F(\theta_s)$ [S1].
    *   Object NeRF $F(\theta_o)$ [S1].
    *   Predefined camera viewpoints $\{C_0, \dots, C_n\}$ [S1].
    *   3D bounding box for the object [S1].
*   **Outputs:**
    *   Final composited image $I_v$ for viewpoint $v$ [S1].

**Data Structures:**
*   **Scene Buffers:** A pre-rendered sequence of RGB-D images $\{(S_0, D_0), \dots, (S_n, D_n)\}$ corresponding to viewpoints $\{C_0, \dots, C_n\}$ [S1]. $S_v$ is the RGB scene image, $D_v$ is the scene depth map [S1].
*   **Object Buffers:** Rendered RGB image $G_v$ and opacity map $O_v$ for the object [S1].

**Steps:**

1.  **Pre-render Scene:** Generate RGB-D images for the established scene NeRF $F(\theta_s)$ at all predefined viewpoints $\{C_0, \dots, C_n\}$ [S1]. Store as $\{(S_v, D_v)\}$ [S1].
2.  **Cast Rays:** For a target viewpoint $v$, cast rays $r_i$ through the image plane [S1].
3.  **Sample Object:** Sample query points $q_k$ along each ray $r_i$ strictly within the object's 3D bounding box [S1].
4.  **Compute Transmittance:** Calculate the transmittance $\tau_k$ to the $k$-th query point using the volume density $\delta_t$ and step size $\Delta_t$ [S1]:
    $$ \tau_k = \exp\left(-\sum_{t=1}^{k-1} \delta_t \Delta_t\right) $$
    where $\delta_t$ is the volume density and $\Delta_t$ is the distance between adjacent query points [S1].
5.  **Render Object RGB:** Compute the object's RGB value $G_v(i)$ for ray $i$ [S1]:
    $$ G_v(i) = \sum_{k=1}^{K} \tau_k (1 - \exp(-\delta_k \Delta_k)) c_k $$
    where $c_k$ is the RGB color at query point $q_k$ and $K$ is the number of samples [S1].
6.  **Render Object Opacity:** Compute the object's opacity $O_v(i)$ [S1]:
    $$ O_v(i) = \begin{cases} \sum_{k=1}^{K} \tau_k (1 - \exp(-\delta_k \Delta_k)), & \text{if } r_i \in \text{box} \text{ and } D_{box}(i) \le D_v(i) \\ 0, & \text{otherwise} \end{cases} $$
    where $D_{box}(i)$ is the depth of the object's front-most intersection along ray $i$ [S1].
7.  **Composite:** Blend the object and scene using the opacity map [S1]:
    $$ I_v = G_v \cdot O_v + S_v \cdot (1 - O_v) $$

**Parameters:**
*   Number of samples $K$: Not given in the sources.
*   Step size $\Delta_k$: Determined by the sampling strategy within the bounding box [S1].

**Edge Cases and Pitfalls:**
*   **No Intersection:** If ray $r_i$ does not intersect the 3D bounding box ($r_i \notin \text{box}$), set $O_v(i) = 0$ [S1].
*   **Occlusion:** If the object is occluded by the scene foreground ($D_{box}(i) > D_v(i)$), set $O_v(i) = 0$ to preserve the original scene content [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compositional_rendering(scene_nerf, object_nerf, viewpoint_v, scene_buffers, object_bbox):
    """
    Renders the composited image for a specific viewpoint.
    
    Args:
        scene_nerf: Pre-trained NeRF for the background scene.
        object_nerf: NeRF for the object to be inserted.
        viewpoint_v: Camera index or parameters for the current view.
        scene_buffers: Dictionary mapping viewpoint index to (S_v, D_v) RGB-D images.
        object_bbox: 3D bounding box defining the object's spatial extent.
        
    Returns:
        I_v: The final composited RGB image.
    """
    # Step 1: Retrieve pre-rendered scene buffers
    S_v, D_v = scene_buffers[viewpoint_v]
    H, W = S_v.shape[:2]
    
    # Initialize output buffers
    G_v = np.zeros((H, W, 3))
    O_v = np.zeros((H, W))
    
    # Get camera rays for viewpoint_v
    rays = get_rays(viewpoint_v, H, W)
    
    for i in range(H * W):
        r = rays[i]
        
        # Step 2 & 3: Sample query points inside the 3D bounding box
        # Determine intersection of ray r with object_bbox
        t_min, t_max = intersect_box(r, object_bbox)
        
        if t_min is None or t_max is None:
            # Ray does not intersect box
            O_v[i] = 0.0
            continue
            
        # Sample points along the ray within the box
        # Assume K samples are distributed between t_min and t_max
        K = 128 # Typical value, not specified in source
        t_values = np.linspace(t_min, t_max, K)
        q_points = r.origin + r.direction * t_values[:, np.newaxis]
        
        # Step 4: Compute Transmittance
        # Calculate density and step size for each segment
        # Note: delta_t is distance between adjacent points
        deltas_t = np.diff(t_values, prepend=t_values[0]) # Approximation for first point
        
        # Query densities from object_nerf
        # densities[k] corresponds to point q_points[k]
        densities = object_nerf.query_density(q_points)
        
        # Calculate transmittance tau_k for each sample k
        # tau_k = exp(-sum_{t=1}^{k-1} delta_t * density_t)
        # We need cumulative sum of delta * density
        attenuation = densities * deltas_t
        cumulative_attenuation = np.cumsum(attenuation)
        tau = np.exp(-np.concatenate([[0], cumulative_attenuation[:-1]]))
        
        # Step 5: Render Object RGB
        # Query colors from object_nerf
        colors = object_nerf.query_color(q_points)
        
        # Weight for each sample: tau_k * (1 - exp(-delta_k * density_k))
        # Note: delta_k in formula refers to step size for sample k
        # Using deltas_t where deltas_t[k] is the length of segment ending at k?
        # Standard volume rendering: weight_k = tau_k * (1 - exp(-density_k * delta_k))
        # Here delta_k is distance between q_k and q_{k+1} or similar. 
        # Based on formula: sum_{k=1}^K tau_k (1 - exp(-delta_k delta_k)) c_k
        # Let's assume deltas_t[k] is the step size associated with sample k.
        # In standard implementation, delta_k is often the distance to the next point.
        # For the last point, delta is 0 or handled separately.
        # Let's use the distance between points.
        step_sizes = np.diff(t_values, append=t_values[-1]) # Distance to next point
        
        weights = tau * (1 - np.exp(-densities * step_sizes))
        G_v_flat = np.sum(weights[:, np.newaxis] * colors, axis=0)
        
        # Step 6: Render Object Opacity
        # Check occlusion against scene depth
        # D_box(i) is the depth of the object's front intersection
        # D_v(i) is the scene depth at pixel i
        # We need to map ray index i to pixel coordinates to get D_v[i]
        u, v = np.unravel_index(i, (H, W))
        D_v_pixel = D_v[v, u]
        
        # Depth of object front along ray
        D_box_i = t_min # Assuming ray origin is at camera, t is depth
        
        if D_box_i <= D_v_pixel:
            O_v_flat = np.sum(weights, axis=0)
        else:
            O_v_flat = 0.0
            
        G_v_flat = G_v_flat.reshape(3)
        G_v[v, u] = G_v_flat
        O_v[v, u] = O_v_flat
        
    # Step 7: Composite
    # I_v = G_v * O_v + S_v * (1 - O_v)
    # O_v is scalar, S_v and G_v are RGB
    I_v = G_v * O_v[:, :, np.newaxis] + S_v * (1 - O_v)[:, :, np.newaxis]
    
    return I_v
```

## Operating characteristics
*   **Time Complexity:** The method reduces computational load by pre-rendering the static scene [S1]. The per-frame cost is dominated by rendering the object NeRF and the final image composition [S1].
*   **Memory Complexity:** Requires storing pre-rendered RGB-D images for all predefined viewpoints $\{C_0, \dots, C_n\}$ [S1].
*   **Strengths:** Decouples object and scene rendering, enhancing flexibility [S1]. Handles 3D occlusions effectively by comparing depths in 3D space before compositing in image space [S1]. Plug-and-play compatible with various scene pre-training methods [S1].
*   **Limitations:** Requires predefined camera viewpoints for pre-rendering [S1]. Limited to viewpoints within the predefined set $\{C_0, \dots, C_n\}$ during optimization [S1].
*   **When to use:** When inserting generated objects into a fixed NeRF scene for VR content creation [S1].
*   **When not to use:** When dynamic scene changes or arbitrary camera viewpoints outside the predefined set are required [S1].

## Background
Compositional Rendering was introduced in the GO-NeRF paper [S1]. It is designed for virtual reality content creation, allowing users to generate and integrate objects into 3D scenes using a user-friendly interface [S1]. The method leverages the stability of pre-trained NeRFs and adds a compositional layer to handle object insertion [S1].

## Sources
[S1] GO-NeRF: Generating Objects in Neural Radiance Fields for Virtual Reality Content Creation, https://doi.org/10.1109/tvcg.2025.3549558, fetched 2026-10-29
