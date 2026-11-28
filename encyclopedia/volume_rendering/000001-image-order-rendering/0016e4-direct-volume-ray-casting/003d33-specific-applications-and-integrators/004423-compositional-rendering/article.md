# Compositional Rendering

Also called: GO-NeRF Compositional Rendering, GO-NeRF Rendering

## Overview
Compositional Rendering is a method for integrating newly generated 3D objects into an existing Neural Radiance Field (NeRF) scene by decoupling the rendering of the object from the background scene [S1]. It solves the problem of occlusion-aware compositing in image-order rendering by generating a 3D-aware opacity map for the new object and blending it with pre-rendered background RGB-D images [S1]. It differs from standard Direct Volume Ray Casting [19-direct-volume-ray-casting.md] by performing the final alpha-blending in image space using a pre-computed scene cache, rather than accumulating the entire scene in a single ray-casting pass [S1].

## How it works
**Inputs:**
*   Pre-rendered background RGB-D image sequences $\{(S_0, D_0), \dots, (S_n, D_n)\}$ for a set of camera viewpoints $\{C_0, \dots, C_n\}$ [S1].
*   A trained object NeRF $F(\theta_o)$ defining volume density $\delta$ and RGB color $c$ [S1].
*   A 3D bounding box defining the spatial extent of the object [S1].
*   A target camera viewpoint $v \in \{C_0, \dots, C_n\}$ [S1].

**Outputs:**
*   Final composited RGB image $I_v$ [S1].
*   Intermediate object RGB image $G_v$ [S1].
*   Intermediate object opacity map $O_v$ [S1].

**Data Structures:**
*   **Background Cache:** A list of tuples, where each tuple contains an RGB image $S_v$ and a depth image $D_v$ for each viewpoint $v$ [S1].
*   **Object NeRF:** A neural network $F(\theta_o)$ that takes a 3D coordinate $q$ and viewing direction as input and outputs volume density $\delta$ and RGB color $c$ [S1].
*   **Bounding Box:** Defined by minimum and maximum coordinates $(x_{min}, y_{min}, z_{min})$ and $(x_{max}, y_{max}, z_{max})$ [S1].

**Steps:**

1.  **Pre-render Background:** For each camera viewpoint $v$ in the predefined set $\{C_0, \dots, C_n\}$, render the background scene using the established scene NeRF $F(\theta_s)$ to generate the RGB image $S_v$ and depth image $D_v$. Store these as the background cache [S1].
2.  **Ray Casting:** For the target viewpoint $v$, cast a ray $r_i$ for each pixel $i$ in the image plane [S1].
3.  **Sampling:** Sample $K$ query points $q_k$ along the ray $r_i$ within the 3D bounding box of the object [S1]. Let $\Delta_k$ be the distance between adjacent query points $q_k$ and $q_{k+1}$ along the ray [S1].
4.  **Query NeRF:** For each query point $q_k$, query the object NeRF $F(\theta_o)$ to obtain the volume density $\delta_k$ and RGB color $c_k$ [S1].
5.  **Calculate Transmittance:** Compute the transmittance $\tau_k$ at sample $k$, representing the probability that a ray passes through the volume up to sample $k$ without being absorbed or scattered [S1]:
    $$ \tau_k = \exp\left(-\sum_{t=1}^{k-1} \delta_t \Delta_t\right) $$
    where $\tau_1 = 1$ [S1].
6.  **Calculate Object RGB:** Compute the accumulated RGB value $G_v(i)$ for pixel $i$ by summing the contribution of each sample weighted by its opacity contribution and transmittance [S1]:
    $$ G_v(i) = \sum_{k=1}^{K} \tau_k (1 - \exp(-\delta_k \Delta_k)) c_k $$
7.  **Calculate Object Opacity:** Compute the accumulated opacity $O_v(i)$ for pixel $i$. This step includes occlusion logic against the pre-rendered background depth $D_v(i)$ [S1]:
    $$ O_v(i) = \begin{cases} \sum_{k=1}^{K} \tau_k (1 - \exp(-\delta_k \Delta_k)), & \text{if } r_i \in \text{box} \text{ AND } D_{box}(i) \le D_v(i) \\ 0, & \text{otherwise} \end{cases} $$
    Here, $D_{box}(i)$ is the depth of the intersection of ray $r_i$ with the 3D bounding box [S1]. The condition $D_{box}(i) \le D_v(i)$ ensures that if the object's bounding box is behind the background scene's geometry, the object is treated as occluded [S1].
8.  **Composite:** Blend the rendered object $G_v$ and the pre-rendered background $S_v$ using the opacity map $O_v$ to produce the final image $I_v$ [S1]:
    $$ I_v = G_v \cdot O_v + S_v \cdot (1 - O_v) $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compositional_rendering(viewpoint_idx, background_cache, object_nerf, bounding_box, num_samples):
    """
    Renders the final composited image for a specific viewpoint.
    
    Args:
        viewpoint_idx: Index of the camera viewpoint in the predefined set.
        background_cache: List of (S_v, D_v) tuples for all viewpoints.
        object_nerf: Function/Model F(theta_o) returning (delta, color) for a batch of points.
        bounding_box: Tuple ((x_min, y_min, z_min), (x_max, y_max, z_max)).
        num_samples: Number of query points K along the ray.
        
    Returns:
        I_v: Final composited RGB image as a flat list of [R, G, B] values.
    """
    S_v, D_v = background_cache[viewpoint_idx]
    height, width = len(S_v), len(S_v[0])
    I_v = [[0.0, 0.0, 0.0] for _ in range(height * width)]
    
    origins, directions = generate_rays(viewpoint_idx)
    
    for i in range(height * width):
        o = origins[i]
        d = directions[i]
        
        # Ray-box intersection (slab method)
        t_min, t_max = intersect_ray_box(o, d, bounding_box)
        
        if t_min is None:
            I_v[i] = S_v[i]
            continue
            
        # Sample points uniformly between t_min and t_max
        t_vals = [t_min + (t_max - t_min) * (k + 0.5) / num_samples for k in range(num_samples)]
        points = [[o[j] + d[j] * t for j in range(3)] for t in t_vals]
        
        deltas, colors = object_nerf(points)
        
        # Compute distances between adjacent samples (Delta_k)
        dists = [t_vals[k+1] - t_vals[k] for k in range(num_samples - 1)]
        dists.insert(0, dists[0] if dists else 0.0)
        
        # Accumulate transmittance and opacity
        cum_density = 0.0
        G_v_pixel = [0.0, 0.0, 0.0]
        O_v_pixel = 0.0
        
        for k in range(num_samples):
            dist = dists[k]
            alpha_k = 1.0 - math.exp(-deltas[k] * dist)
            tau_k = math.exp(-cum_density)
            
            for c in range(3):
                G_v_pixel[c] += tau_k * alpha_k * colors[k][c]
            O_v_pixel += tau_k * alpha_k
            
            cum_density += deltas[k] * dist
            
        # Occlusion check: D_box is t_min (depth of front intersection)
        if t_min > D_v[i]:
            O_v_pixel = 0.0
            
        for c in range(3):
            I_v[i][c] = G_v_pixel[c] * O_v_pixel + S_v[i][c] * (1.0 - O_v_pixel)
            
    return I_v

def generate_rays(viewpoint_idx):
    # [G] Standard pinhole camera ray generation. Returns list of origins and directions.
    # Implementation depends on camera intrinsics and extrinsics for viewpoint_idx.
    pass

def intersect_ray_box(origin, direction, box):
    # [G] Standard AABB intersection using slab method.
    # box = ((x_min, y_min, z_min), (x_max, y_max, z_max))
    t_min, t_max = 0.0, float('inf')
    for d in range(3):
        inv_d = 1.0 / direction[d] if direction[d] != 0 else float('inf')
        t1 = (box[0][d] - origin[d]) * inv_d
        t2 = (box[1][d] - origin[d]) * inv_d
        t_min = max(t_min, min(t1, t2))
        t_max = min(t_max, max(t1, t2))
    if t_min > t_max:
        return None, None
    return t_min, t_max
```

**Parameters:**
*   `num_samples` ($K$): Number of query points along the ray. Not given in the sources.
*   `bounding_box`: Defines the 3D extent of the object [S1].

**Edge Cases and Pitfalls:**
*   **No Intersection:** If the ray $r_i$ does not intersect the 3D bounding box, the opacity $O_v(i)$ is set to 0, and the background $S_v$ is used directly [S1].
*   **Occlusion:** If the depth of the bounding box intersection $D_{box}(i)$ is greater than the background depth $D_v(i)$, the object is considered occluded by the scene foreground. In this case, $O_v(i)$ is set to 0, and the background $S_v$ is used [S1].
*   **Precision:** The exponential function in transmittance calculation can suffer from underflow for high densities or large distances. [G] Standard practice is to clamp densities or use log-space accumulation.

## Operating characteristics
*   **Time Complexity:** The method reduces computational load by pre-rendering the background scene [S1].
*   **Memory Complexity:** Requires storing the pre-rendered RGB-D image sequences for all viewpoints $\{C_0, \dots, C_n\}$ [S1].
*   **Strengths:** Decouples object and scene rendering, enhancing flexibility [S1]. Handles occlusions effectively using 3D-aware opacity maps [S1]. Plug-and-play compatibility with various scene pre-training methods [S1].
*   **Limitations:** Requires pre-rendering the background for all predefined viewpoints [S1]. Limited to the predefined set of camera viewpoints [S1].
*   **When to Use:** When integrating generated objects into a static NeRF scene with known viewpoints, especially for VR content creation where specific viewpoints are important [S1].

## Background
Compositional Rendering was introduced in the GO-NeRF paper [S1]. It is designed for virtual reality content creation, allowing users to generate and integrate objects into 3D scenes using a user-friendly interface [S1]. The method leverages the decoupling of object and scene rendering to simplify the process for casual users [S1].

## Sources
[S1] GO-NeRF: Generating Objects in Neural Radiance Fields for Virtual Reality Content Creation, https://doi.org/10.1109/tvcg.2025.3549558, fetched 2026-10-29
