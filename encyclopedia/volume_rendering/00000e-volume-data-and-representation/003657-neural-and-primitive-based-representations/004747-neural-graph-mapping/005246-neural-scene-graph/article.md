# Neural Scene Graph

Also called: NSG

## Overview
Neural Scene Graphs (NSG) represent dynamic scenes as a hierarchical directed acyclic graph that decomposes the environment into static background and dynamic object components, enabling novel view synthesis and dynamic scene reconstruction [S1]. Unlike monolithic neural radiance fields, NSG isolates rigidly connected dynamic components into individual nodes with shared class weights and latent codes, allowing for pose-dependent appearance changes while maintaining volumetric consistency [S1].

## How it works

### Inputs and Outputs
**Inputs:**
*   Camera intrinsics $K \in \mathbb{R}^{3 \times 3}$ and transformation $T_c^W$ (camera to world frame) [S1].
*   3D tracking data providing transformations $M_{uv}$ for dynamic objects [S1].
*   Ground truth images for training [S1].

**Outputs:**
*   Rendered pixel colors $\hat{C}(r)$ for rays $r$ [S1].
*   Optimized scene graph $S$ with trained representation networks and latent codes [S1].

### Data Structures
The scene is defined as a directed acyclic graph $S = (W, C, F, L, E)$ [S1]:
*   $C$: Leaf node representing the camera [S1].
*   $F$: Set of leaf nodes for representation models, $F = F_{\theta_{bckg}} \cup \{F_{\theta_c}\}_{c=1}^{N_{obj}}$, where $F_{\theta_{bckg}}$ is the static background model and $F_{\theta_c}$ are dynamic object models [S1].
*   $L$: Set of leaf nodes for latent object codes, $L = \{l_o\}_{o=1}^{N_{obj}}$, assigning a latent vector to each dynamic object [S1].
*   $E$: Edges defining the hierarchy and transformations (e.g., object poses) [S1].
*   $W$: Implicitly defined by the graph structure and node functions [S1].

### Parameters
*   $N_s$: Number of planes for static background sampling. Typical value: 6 [S1].
*   $N_d$: Number of equidistant sampling points per dynamic object intersection. Typical value: 7 [S1].
*   $\sigma$: Standard deviation for the spherical covariance prior distribution over latent codes [S1].
*   $s_o = [L_o, H_o, W_o]$: Bounding box size of object $o$ [S1].

### Algorithm Steps

1.  **Ray Generation**: Generate rays $r = o + td$ through each pixel of the film size $H \times W$ using camera intrinsics $K$ and transformation $T_c^W$ [S1].
2.  **Static Background Sampling**:
    *   Define $N_s$ planes parallel to the initial camera pose $T_{c,0}^W$, equispaced between near clip $d_n$ and far clip $d_f$ [S1].
    *   For each ray $r$, calculate intersections $\{t_i\}_{i=1}^{N_s}$ with these planes [S1].
3.  **Dynamic Object Intersection**:
    *   For each ray, check intersections with dynamic nodes using an Axis-Aligned Bounding Box (AABB) ray intersection test in the object's local frame [S1].
    *   Compute entrance and exit points $(t_{o,1}, t_{o,N_d})$ for each dynamic node $o$ intersected by the ray [S1].
4.  **Dynamic Object Sampling**:
    *   For each pair of entrance and exit points, sample $N_d$ equidistant quadrature points $t_{o,n}$:
        $$ t_{o,n} = \frac{(n-1)(t_{o,N_d} - t_{o,1})}{N_d - 1} + t_{o,1} $$
        for $n = [1, N_d]$ [S1].
5.  **Point Transformation**:
    *   Transform global points $x$ to local object coordinates $x_o$ using the scaling matrix $S_o$ and transformation matrix $T_o$:
        $$ x_o = S_o T_o x $$
    *   The scaling matrix is $S_o = \text{diag}(1/s_o)$, where $s_o = [L_o, H_o, W_o]$ is the bounding box size, ensuring size-independent learning [S1].
    *   Local directions $d_o$ are derived from the ray direction [S1].
6.  **Density and Color Computation**:
    *   **Static Background**: Compute density $y(x)$ and volumetric density $\sigma(x)$ using the first stage, then color $c(x)$ using the second stage:
        $$ [y(x), \sigma(x)] = F_{\theta_{bckg},1}(\gamma_x(x), l_o) $$
        $$ c(x) = F_{\theta_{bckg},2}(\gamma_d(d), y(x)) $$
        where $\gamma_x$ and $\gamma_d$ are Fourier encodings [S1].
    *   **Dynamic Objects**: Compute density $\sigma(x_o)$ and color $c(x_o)$ using the object model. The pose $p_o$ is included only in the color computation to ensure volumetric consistency:
        $$ [\sigma(x_o), y(x_o)] = F_{\theta_{c},1}(\gamma_x(x_o), l_o) $$
        $$ c(x_o) = F_{\theta_{c},2}(\gamma_d(d_o), y(x_o), p_o) $$
        where $p_o$ is the global location of the object [S1].
7.  **Ordering**: Order all sampling points and model outputs along the ray $r(t)$ such that $t_{i-1} < t_i$ [S1].
8.  **Rendering**: Compute the transmitted color $\hat{C}(r)$ using numerical quadrature:
    $$ \hat{C}(r) = \sum_{i=1}^{N_s + m_j N_d} T_i \alpha_i c_i $$
    where $T_i = \exp(-\sum_{k=1}^{i-1} \sigma_k \delta_k)$, $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$, and $\delta_i = t_{i+1} - t_i$ [S1].
9.  **Optimization**:
    *   Minimize the loss function $\mathcal{L}$ over a batch of rays $R$:
        $$ \mathcal{L} = \sum_{r \in R} \|\hat{C}(r) - C(r)\|_2^2 + \frac{1}{2\sigma^2} \|z\|_2^2 $$
        where $C(r)$ is the ground truth color and $z$ represents the latent codes [S1].
    *   Backpropagate gradients to each trainable node in $L$ and $F$ intersecting with the rays in $R$ [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def fourier_encoding(x, freq_bands=10):
    """
    Standard Fourier feature mapping.
    x: input tensor of shape [..., D]
    Returns: tensor of shape [..., 2 * D * freq_bands]
    """
    # Implementation detail: [G] Standard Fourier features as per [37] in source.
    # Scales input by frequencies 2^0 to 2^(freq_bands-1).
    # Concatenates sin and cos of scaled inputs.
    pass

def compute_static_background(x, d, latent_code, model_bckg):
    """
    Computes density and color for static background.
    """
    gamma_x = fourier_encoding(x)
    gamma_d = fourier_encoding(d)
    
    # Stage 1: Density
    y, sigma = model_bckg.stage1(gamma_x, latent_code)
    
    # Stage 2: Color
    color = model_bckg.stage2(gamma_d, y)
    
    return color, sigma

def compute_dynamic_object(x_global, d_global, pose, latent_code, model_obj):
    """
    Computes density and color for dynamic object.
    Volumetric density is pose-independent; color is pose-dependent.
    """
    # Transform to local frame
    # S_o = diag(1 / bounding_box_size)
    # x_local = S_o @ T_o @ x_global
    x_local = transform_to_local(x_global, pose)
    d_local = transform_direction(d_global, pose)
    
    gamma_x = fourier_encoding(x_local)
    gamma_d = fourier_encoding(d_local)
    gamma_p = fourier_encoding(pose) # Pose encoded for color
    
    # Stage 1: Density (independent of pose)
    y, sigma = model_obj.stage1(gamma_x, latent_code)
    
    # Stage 2: Color (dependent on pose)
    color = model_obj.stage2(gamma_d, y, gamma_p)
    
    return color, sigma

def render_ray(ray_origin, ray_direction, camera_intrinsics, scene_graph, models):
    """
    Renders a single ray through the Neural Scene Graph.
    """
    # 1. Static Background Sampling
    # Ns planes parallel to initial camera pose
    plane_intersections = calculate_plane_intersections(ray_origin, ray_direction, scene_graph.planes)
    
    static_points = []
    for t in plane_intersections:
        x = ray_origin + t * ray_direction
        color, sigma = compute_static_background(x, ray_direction, scene_graph.background_latent, models['bckg'])
        static_points.append((t, color, sigma))
        
    # 2. Dynamic Object Sampling
    dynamic_points = []
    for obj in scene_graph.dynamic_objects:
        # AABB-Ray intersection in local frame
        intersections = aabb_ray_intersect(ray_origin, ray_direction, obj.bbox, obj.pose)
        
        if intersections is not None:
            t_start, t_end = intersections
            # Sample Nd equidistant points
            for n in range(1, obj.Nd + 1):
                t = (n - 1) * (t_end - t_start) / (obj.Nd - 1) + t_start
                x = ray_origin + t * ray_direction
                color, sigma = compute_dynamic_object(x, ray_direction, obj.pose, obj.latent_code, models['obj'])
                dynamic_points.append((t, color, sigma))
                
    # 3. Order all points
    all_points = static_points + dynamic_points
    all_points.sort(key=lambda p: p[0])
    
    # 4. Volume Rendering
    total_transmittance = 1.0
    accumulated_color = np.zeros(3)
    
    for i, (t, color, sigma) in enumerate(all_points):
        if i == 0:
            delta = all_points[1][0] - t
        else:
            delta = all_points[i][0] - all_points[i-1][0]
            
        alpha = 1.0 - np.exp(-sigma * delta)
        transmittance = total_transmittance * np.exp(-sigma * delta)
        
        accumulated_color += transmittance * alpha * color
        total_transmittance *= (1.0 - alpha)
        
    return accumulated_color

def train_scene_graph(scene_graph, models, rays, ground_truth_colors, optimizer):
    """
    Optimizes the scene graph and models.
    """
    predicted_colors = []
    for ray in rays:
        pred = render_ray(ray.origin, ray.direction, ray.camera, scene_graph, models)
        predicted_colors.append(pred)
        
    predicted_colors = np.array(predicted_colors)
    
    # Loss: Squared error + L2 regularization on latent codes
    reconstruction_loss = np.sum((predicted_colors - ground_truth_colors) ** 2)
    
    latent_codes = [obj.latent_code for obj in scene_graph.dynamic_objects]
    latent_loss = 0.0
    for z in latent_codes:
        latent_loss += np.sum(z ** 2)
        
    total_loss = reconstruction_loss + (1.0 / (2 * scene_graph.sigma ** 2)) * latent_loss
    
    # Backpropagate gradients
    optimizer.zero_grad()
    total_loss.backward()
    optimizer.step()
    
    return total_loss
```

### Edge Cases and Pitfalls
*   **Volumetric Consistency**: Object volumetric density must not change based on pose to ensure volumetric consistency; pose is only used for emitted color [S1].
*   **Bounding Box Scaling**: The scaling matrix $S_o$ must be defined as $\text{diag}(1/s_o)$ where $s_o$ is the bounding box size to enable size-independent learning [S1].
*   **Plane Sampling**: The static background uses sparse planes instead of a volume, which limits sampling to 2.5D representation [S1].

## Subtopics

### [Neural Scene Graph Representation and Rendering](14688-neural-scene-graph-representation-and-rendering.md)
A hierarchical scene representation that decomposes scenes into dynamic and static components without additional supervision, where each object is represented as an individual node.

## Operating characteristics
*   **Time Complexity**: Rendering involves ray-plane intersections for static background and AABB-ray intersections for dynamic objects. Sampling is fixed at $N_s$ and $N_d$ points per intersection [S1].
*   **Memory Complexity**: Depends on the number of dynamic objects and the size of the MLPs for background and object models [S1].
*   **Performance**: Optimization for a scene takes about a full day or 350k iterations to converge on a single NVIDIA TITAN Xp GPU [S1].
*   **Strengths**: Hierarchical decomposition allows for dynamic scene reconstruction and novel view synthesis. Shared weights between objects of the same class improve generalization [S1].
*   **Limitations**: Relies on 3D tracking data for transformations [S1]. Not limited to automotive datasets but primarily evaluated on KITTI and Virtual KITTI 2 [S1].

## Background
Neural Scene Graphs were introduced in the paper "Neural Scene Graphs for Dynamic Scenes" [S1]. The method builds on neural radiance fields and sparse plane representations, extending them to handle dynamic scenes by decomposing them into static and dynamic components [S1]. It is used on automotive datasets like KITTI and Virtual KITTI 2 for dynamic scene reconstruction and novel view synthesis [S1].

## Sources
[S1] Neural Scene Graphs for Dynamic Scenes, https://doi.org/10.1109/cvpr46437.2021.00288, fetched 2026-10-26
