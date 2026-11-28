# NeurMiPs

Also called: NeurMiPs (Neural Mixture of Planar Experts), Neural Mixture of Planar Experts, Neural Mixture of Planar Experts for View Synthesis

## Overview
NeurMiPs represents a 3D scene as a mixture of $K$ learnable rectangular planar surfaces, where each surface acts as an independent "expert" predicting view-dependent color and transparency via a local neural radiance field [S1]. It solves the frontal-parallel limitation of traditional Multi-Plane Imagery (MPI) and the blurring artifacts of large-scale NeRFs by combining explicit geometric priors with implicit appearance modeling, offering superior extrapolation capabilities and real-time rendering speeds [S1]. Unlike voxel-based methods that prune unseen regions, NeurMiPs maintains a continuous representation on planes, allowing it to generalize better to novel viewpoints [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A sparse 3D point cloud $\{x_i\}$ from Structure-from-Motion (SfM) [S1]; a set of training images with known camera poses [S1].
*   **Outputs:** Rendered pixel color $c(\mathbf{r})$ and expected depth $d(\mathbf{r})$ for a ray $\mathbf{r}$ [S1].

**Data Structures:**
1.  **Planar Expert Geometry ($s_k$):** A tuple for each expert $k \in \{1, \dots, K\}$ containing:
    *   $p_k \in \mathbb{R}^3$: Center coordinate of the rectangle.
    *   $n_k \in \mathbb{R}^3$: Normalized 3D normal vector of the plane.
    *   $u_k \in \mathbb{R}^3$: Normalized up vector defining the y-axis direction of the plane's local coordinate system.
    *   $w_k, h_k \in \mathbb{R}$: Width and height of the rectangle [S1].
2.  **Student Radiance Field ($f_k$):** A Multi-Layer Perceptron (MLP) for each expert $k$.
    *   **Architecture:** Three fully-connected hidden layers with ReLU activation; final output layer with sigmoid activation [S1].
    *   **Input:** 3D coordinate $\mathbf{x}_k$ on the plane and normalized ray direction $\mathbf{d} \in S^2$. The direction $\mathbf{d}$ is transformed into a higher-dimensional space using high-frequency positional encoding before entering the network [S1].
    *   **Output:** Color $\mathbf{c}_k \in \mathbb{R}^3$ and transparency $\alpha_k \in \mathbb{R}$ [S1].
3.  **Alpha Textures ($A_i$):** Optional pre-baked textures of size $w \times h$ for each plane $i$, storing view-independent alpha values for accelerated inference [S1].

**Numbered Steps:**

1.  **Plane Initialization:**
    Initialize plane parameters $\{s_k\}$ using the coarse 3D point cloud $\{x_i\}$ from SfM. Plane centers $p_k$ are selected via farthest point sampling. Orientations $n_k, u_k$ are initialized as normals estimated from local point sets around $p_k$ [S1]. Optimize the geometry parameters by minimizing the geometry loss $\mathcal{L}_g$:
    $$ \mathcal{L}_g = \sum_i \min_k d(x_i, s_k) + \lambda \sum_k (w_k h_k)^2 $$
    where $d(x_i, s_k)$ is the distance from point $x_i$ to the closest rectangular surface $s_k$, and $(w_k h_k)^2$ is an area regularization term to prevent arbitrarily large rectangles. $\lambda$ is a weighting factor [S1].

2.  **Teacher Model Training:**
    Train a large-capacity standard NeRF (the "teacher") to serve as a knowledge source. Point sampling for the teacher is conducted through the NeurMiPs ray-plane intersection mechanism. Jointly optimize the planar experts' geometric parameters and the teacher's weights using the total loss:
    $$ \mathcal{L}_{total} = \mathcal{L}_g + \mathcal{L}_c $$
    where $\mathcal{L}_c$ is the photometric loss [S1].

3.  **Distillation:**
    Distill knowledge from the teacher to the student experts ($f_k$). For each planar expert, draw random points uniformly from its rectangle and random view directions from the half unit sphere. Update the student network parameters by minimizing the L2 loss between the teacher's output $(\mathbf{c}_{teacher}, \alpha_{teacher})$ and the student's output $(\mathbf{c}_k, \alpha_k)$ for both color and alpha values [S1].

4.  **Fine-Tuning:**
    Fix the plane parameters $\{s_k\}$. Fine-tune the student radiance field models $f_k$ to improve rendering quality by minimizing the L2-photometric loss $\mathcal{L}_c$ between the rendered pixel color and the ground-truth color:
    $$ \mathcal{L}_c = \sum_{\mathbf{r}} \|c(\mathbf{r}) - c_{\text{gt}}(\mathbf{r})\|_2^2 $$
    [S1].

5.  **Ray-Plane Intersection:**
    For a ray $r = \{o, d\}$ (origin $o$, direction $d$) and plane $k$ (normal $n_k$, center $p_k$), compute the intersection point $x_k$:
    $$ x_k = o + \frac{(p_k - o) \cdot n_k}{d \cdot n_k} d $$
    Only keep rectangles that are actually intersected by the ray [S1].

6.  **Radiance Evaluation:**
    For each intersecting planar expert $k$, evaluate the neural radiance function $f_k$ using the intersection point coordinate $x_{p,k}$ (projected onto the plane's local 2D coordinates) and ray direction $d$ to output transparency $\alpha_k$ and color $c_k$ [S1].
    *Note: If Alpha Baking is enabled, retrieve $\alpha_k$ by bilinear interpolation from the baked texture $A_k$ instead of evaluating the network for alpha [S1].*

7.  **Alpha Composition:**
    Collect the set of point samples $\{x_j, c_j, \alpha_j\}$ from all intersecting experts. Sort the points from closest to furthest relative to the eye $o$. Compute the final ray color:
    $$ c(\mathbf{r}) = \sum_j \left( \prod_{i}^{j-1} (1 - \alpha_i) \right) \alpha_j c_j $$
    [S1].

8.  **Early Ray Termination (Optional):**
    If the accumulated transmittance $\prod (1 - \alpha_i)$ is close to zero, stop evaluating further intersecting rectangles to save computation. If the sum of alpha values is smaller than 1, re-normalize the alphas [S1].

9.  **Depth Estimation (Optional):**
    Estimate expected depth $d(\mathbf{r})$ along the ray using alpha composition:
    $$ d(\mathbf{r}) = \sum_j \left( \prod_{i}^{j-1} (1 - \alpha_i) \right) \alpha_j t_j $$
    where $t_j$ is the depth (distance from origin) of sampled point $j$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def ray_plane_intersection(ray_origin, ray_dir, plane_center, plane_normal):
    """
    Computes intersection of ray with infinite plane.
    Returns distance t along ray, or None if parallel/no intersection.
    """
    denom = np.dot(ray_dir, plane_normal)
    if np.isclose(denom, 0.0):
        return None
    t = np.dot(plane_center - ray_origin, plane_normal) / denom
    if t < 1e-6: # Intersection behind ray origin
        return None
    return t

def project_to_plane(point, plane_center, plane_normal, plane_up):
    """
    Projects 3D point to 2D coordinates on the plane.
    Assumes plane_up is orthogonal to plane_normal.
    """
    # Construct local basis
    # plane_right = cross(plane_normal, plane_up) # Not strictly needed if up is fixed
    # However, standard projection uses two orthogonal vectors in the plane.
    # Let's assume plane_up defines Y, and we derive X.
    plane_right = np.cross(plane_normal, plane_up)
    plane_right = plane_right / np.linalg.norm(plane_right)
    
    v = point - plane_center
    x_coord = np.dot(v, plane_right)
    y_coord = np.dot(v, plane_up)
    return np.array([x_coord, y_coord])

def render_ray_neurmiips(ray_origin, ray_dir, planes, student_networks, use_alpha_baking, alpha_textures):
    """
    Renders a single ray using NeurMiPs.
    planes: List of dicts with keys 'center', 'normal', 'up', 'width', 'height'
    student_networks: List of MLP functions f_k
    use_alpha_baking: Boolean
    alpha_textures: List of 2D arrays (w x h) for baked alpha values
    """
    intersections = []
    
    # Step 5: Ray-Plane Intersection
    for k, plane in enumerate(planes):
        t = ray_plane_intersection(ray_origin, ray_dir, plane['center'], plane['normal'])
        if t is not None:
            # Check if intersection point is within the rectangle bounds
            intersection_point = ray_origin + t * ray_dir
            local_coords = project_to_plane(intersection_point, plane['center'], plane['normal'], plane['up'])
            
            # Check bounds [-w/2, w/2] and [-h/2, h/2]
            half_w = plane['width'] / 2.0
            half_h = plane['height'] / 2.0
            
            if -half_w <= local_coords[0] <= half_w and -half_h <= local_coords[1] <= half_h:
                intersections.append({
                    'k': k,
                    't': t,
                    'local_coords': local_coords,
                    'point': intersection_point
                })
    
    # Sort by distance (closest to furthest)
    intersections.sort(key=lambda x: x['t'])
    
    accumulated_transmittance = 1.0
    final_color = np.zeros(3)
    
    # Step 6 & 7: Radiance Evaluation and Alpha Composition
    for item in intersections:
        k = item['k']
        local_coords = item['local_coords']
        
        # Evaluate Radiance
        # Input to network: local_coords (2D) and ray_dir (3D)
        # Note: The paper says input is x_k (3D space coordinate on plane) and d.
        # Usually, for planar experts, the 3D coordinate is mapped to 2D or used directly.
        # The paper states: "evaluate ... using intersection point coordinate x_{p,k}".
        # We assume the network takes the 2D local coordinates or the 3D point.
        # Given "x_k (3D space coordinate on the plane)" in formula 1, we pass the 3D point.
        # However, standard practice for planar sprites often uses 2D. 
        # The paper says: "The function takes the 3D space coordinate xk ... as input".
        # So we pass the 3D intersection point.
        
        input_coord = item['point']
        input_dir = ray_dir
        
        # High-frequency encoding for direction (standard NeRF practice mentioned in paper)
        # Assuming a standard positional encoding function exists
        encoded_dir = positional_encoding(input_dir) 
        
        # Student Network Evaluation
        # f_k takes (x_k, d)
        # We need to implement the MLP structure: 3 hidden layers (ReLU), 1 output (Sigmoid)
        # For brevity, assume student_networks[k] is a callable that handles this
        color, alpha = student_networks[k](input_coord, encoded_dir)
        
        # Alpha Baking (Optional Optimization)
        if use_alpha_baking:
            # Retrieve alpha from texture via bilinear interpolation
            # Map local_coords to texture coordinates [0, 1]
            h, w = alpha_textures[k].shape
            u = (local_coords[0] + planes[k]['width']/2) / planes[k]['width']
            v = (local_coords[1] + planes[k]['height']/2) / planes[k]['height']
            # Bilinear interpolation logic would go here
            alpha = bilinear_interpolate(alpha_textures[k], u, v)
            
        # Early Ray Termination
        if accumulated_transmittance < 1e-4:
            break
            
        # Alpha Composition
        weight = accumulated_transmittance * alpha
        final_color += weight * color
        accumulated_transmittance *= (1 - alpha)
        
    return final_color

def train_neurmiips(planes, student_networks, teacher_network, point_cloud, training_images, optimizer):
    """
    Training loop for NeurMiPs.
    """
    # Hyperparameters
    teacher_epochs = 6000
    distill_epochs = 1500
    finetune_epochs = 2500
    lr = 5e-4
    
    # Step 1: Plane Initialization
    # Optimize plane geometry parameters {s_k} using point cloud
    # Loss: L_g = sum(min_k d(x_i, s_k)) + lambda * sum((w_k h_k)^2)
    optimize_geometry(planes, point_cloud)
    
    # Step 2: Teacher Model Training
    # Train teacher NeRF. Sampling points via NeurMiPs ray-plane intersection.
    # Jointly optimize plane geometry and teacher weights.
    for epoch in range(teacher_epochs):
        for image in training_images:
            # Render using current planes and teacher
            # Loss: L_total = L_g + L_c
            loss = compute_loss_teacher(planes, teacher_network, image)
            optimizer.step(loss)
            
    # Step 3: Distillation
    # Fix teacher? No, distill from teacher to students.
    # Draw random points from rectangles and random directions.
    for epoch in range(distill_epochs):
        for k in range(len(planes)):
            # Sample points uniformly from rectangle k
            # Sample directions from half unit sphere
            # Minimize L2 loss between teacher output and student output (color + alpha)
            loss_distill = compute_distillation_loss(planes[k], student_networks[k], teacher_network)
            optimizer.step(loss_distill)
            
    # Step 4: Fine-Tuning
    # Fix plane parameters
    for epoch in range(finetune_epochs):
        for image in training_images:
            # Render using students
            # Minimize L_c (photometric loss)
            loss_finetune = compute_photometric_loss(planes, student_networks, image)
            optimizer.step(loss_finetune)
```

**Parameters:**
*   $K$: Number of planar experts. Typical values: 500 for Replica dataset, 1000 for Tanks & Temples dataset [S1].
*   Optimizer: Adam with learning rate $5 \times 10^{-4}$ [S1].
*   Training Epochs: Teacher model: 6,000 epochs; Distillation: 1,500 epochs; Fine-tuning: 2,500 epochs [S1].
*   $\lambda$: Weighting factor for area regularization in $\mathcal{L}_g$. Value not specified in sources [S1].

**Edge Cases and Pitfalls:**
*   **Finite Plane Size:** Planes may not cover all regions in extreme viewpoints, inducing higher depth error for scenes modeled by the background. This can be resolved by adopting multiple-layer boxes or incorporating techniques from NeRF++ for unbounded scenes [S1].
*   **SfM Dependency:** The method relies heavily on SfM point clouds for plane initialization. If sparse point clouds are noisy or unavailable, performance degrades significantly [S1].
*   **Unbounded Scenes:** NeurMiPs currently cannot handle unbounded scenes natively. A solution involves modeling background texture with non-Euclidean coordinates [S1].
*   **Alpha Baking Artifacts:** Applying alpha baking directly during inference might bring a minor rendering performance drop or quality drop due to discretization of the continuous radiance field [S1].

## Operating characteristics
*   **Time Complexity:** Rendering is efficient due to closed-form ray-plane intersection and the fact that each ray hits only a handful of planes. Custom CUDA kernels fuse network evaluation, allowing parallel rendering of all experts [S1].
*   **Memory Complexity:** Significantly lower memory footprint compared to voxel-based approaches (e.g., PlenOctrees, KiloNeRF) because it uses compact planar parameters and small MLPs [S1]. Alpha baking increases memory usage for textures [S1].
*   **Measured Performance:**
    *   **Replica Dataset:** Achieves 19.16 frames per second (FPS) on a single TITAN RTX using customized CUDA kernels [S1].
    *   **Tanks & Temples (Truck Scene):** Achieves 976 FPS at 1920x1080 resolution on a single RTX 3090 desktop using texture baking [S1].
    *   **Quality (Tanks & Temples):** PSNR 28.46, SSIM 0.908, LPIPS 0.089 [S1].
    *   **Quality (Replica):** PSNR 30.80, SSIM 0.900, LPIPS 0.088 [S1].
*   **Strengths:** More sample-efficient than volume rendering; better extrapolation than voxel-based methods; captures detailed textures on planar surfaces with sharp boundaries; avoids "fog-like" artifacts of NeRF and "stack-of-cards" effects of MPI [S1].
*   **Limitations:** Cannot handle unbounded scenes without modification; performance degrades with poor/no SfM data; finite plane size causes depth errors in extreme viewpoints [S1].

## Background
NeurMiPs (Neural Mixture of Planar Experts) was introduced in the paper "NeurMiPs: Neural Mixture of Planar Experts for View Synthesis" [S1]. It builds upon the success of neural radiance fields (NeRF) and multi-plane imagery (MPI), aiming to combine the geometric flexibility of neural rendering with the efficiency of explicit planar representations. The method was developed to address the limitations of existing approaches in large-scale scenes and view extrapolation [S1].

## Sources
[S1] NeurMiPs: Neural Mixture of Planar Experts for View Synthesis, https://doi.org/10.1109/cvpr52688.2022.01525, fetched 2026-10-26
