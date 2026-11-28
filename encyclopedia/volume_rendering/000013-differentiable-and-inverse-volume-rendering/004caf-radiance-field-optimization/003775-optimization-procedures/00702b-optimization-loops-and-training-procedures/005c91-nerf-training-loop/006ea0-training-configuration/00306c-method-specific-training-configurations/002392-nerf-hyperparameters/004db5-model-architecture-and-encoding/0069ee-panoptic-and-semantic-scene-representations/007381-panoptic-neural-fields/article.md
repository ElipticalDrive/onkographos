# Panoptic Neural Fields

Also called: MARS, PNF / LANe / MARS, Panoptic Neural Fields (PNF) Model Architecture, Panoptic Neural Fields (PNF) Rendering & Compositing

## Overview
Panoptic Neural Fields (PNF) is a semantic object-aware neural scene representation that decomposes a dynamic 3D scene into distinct "things" (moving objects) and "stuff" (static background), representing each with separate MLP-based neural functions [S2]. Unlike sibling methods like [Panoptic Radiance Field](15849-panoptic-radiance-field.md) or [Category-Level Semantic NeRF](20302-category-level-semantic-nerf.md) which may share MLPs or lack dynamic object handling, PNF uses instance-specific fully weight-encoded MLPs for each object, allowing it to model dynamic scenes and specific instance details without latent code bottlenecks [S2]. It learns a 4D panoptic radiance field from RGB images alone, querying color, density, instance, and category labels for any 3D point over time [S2].

## How it works
**Inputs and Outputs**
*   **Inputs:** A sequence of RGB images, predicted camera parameters, 2D semantic segmentations (pseudo-supervision), and 3D object detections with oriented bounding boxes and category labels [S2]. No 3D ground-truth or semantic labels are required during training [S2].
*   **Outputs:** For a query $(\mathbf{x}, \mathbf{d}, t)$, the model outputs color $\mathbf{c}$, volume density $\sigma$, semantic label (category), and instance label [S2].

**Data Structures**
1.  **Stuff Field:** A single MLP (or two for large scenes: foreground/background) representing the static background [S2].
    *   **Architecture:** 8 hidden layers, width 256 [S2].
    *   **Input:** Encoded position $\mathbf{x}$ and view direction $\mathbf{d}$ [S2].
    *   **Outputs:** Density $\sigma_s$, color $\mathbf{c}_s$, semantic logits $\mathbf{l}_s$ [S2].
2.  **Thing Fields:** A set of separate MLPs, one per object instance $k$ [S2].
    *   **Architecture:** 4 hidden layers, width 128 [S2].
    *   **Input:** Encoded object coordinates $\mathbf{x}'$ and view direction $\mathbf{d}$ [S2].
    *   **Outputs:** Density $\sigma_k$, color $\mathbf{c}_k$, semantic logits $\mathbf{l}_k$ [S2].
3.  **Object Track $T_k$:** Parameterizes the pose of object $k$ over time [S2].
    *   **Parameters:** For each discrete timestamp, a rotation matrix $\mathbf{R} \in \mathbb{R}^{3\times3}$, translation vector $\mathbf{t} \in \mathbb{R}^3$, and time-invariant box extent $\mathbf{s} \in \mathbb{R}^3$ [S2].
    *   **Interpolation:** For arbitrary real-valued timestamps, track steps are interpolated to determine the coordinate frame [S2].

**Positional Encoding**
The input coordinates are mapped using hierarchical positional encoding $\gamma(\cdot)$ [S1].
*   **Stuff Position:** $L=10$ frequencies [S2].
*   **Thing Position:** $L=6$ frequencies in object coordinate space [S2].
*   **View Direction:** $L=2$ frequencies, gradually activated over optimization [S2].
*   **Formula:** For scalar $p$, $\gamma(p) = (\sin(2^0 \pi p), \cos(2^0 \pi p), \dots, \sin(2^{L-1} \pi p), \cos(2^{L-1} \pi p))$ [S1]. Applied independently to each component of $\mathbf{x}$ and $\mathbf{d}$ [S1].

**Panoptic-Radiance Field Composition**
The final field aggregates contributions from bounding box hits, defaulting to stuff if no intersection occurs [S2]. For any output channel $f$ (color, density, semantic, instance):
$$ f(\mathbf{x} | \theta) = \mathbb{1}_S(\mathbf{x}) f_s(\mathbf{x} | \theta) + \sum_k f_k(T_k^{-1} \mathbf{x} | \theta) $$
where $\mathbb{1}_S(\mathbf{x}) = 1$ if $\mathbf{x}$ intersects no bounding boxes, $f_s$ is the stuff field, and $f_k$ is the thing field for instance $k$ [S2]. $T_k^{-1}$ transforms world coordinates to the object's local bounding box frame at time $t$ [S2].

**Training Steps**
1.  **Initialization:**
    *   Run off-the-shelf algorithms to predict camera poses, 2D semantic segmentations, and 3D object detections with bounding boxes [S2].
    *   Initialize Thing MLP weights using category-specific meta-learned initialization (via FedAvg on ShapeNet data) or biased density activation initialization [S2].
    *   Initialize Stuff MLP weights [S2].
2.  **Ray Sampling:**
    *   Generate rays from camera intrinsics/extrinsics [S1].
    *   Sample 1024 points per ray using stratified sampling (no hierarchical sampling) [S2].
    *   Transform sampled world positions $\mathbf{x}$ to object local coordinates $\mathbf{x}' = T_k^{-1} \mathbf{x}$ for each intersecting bounding box [S2].
3.  **Forward Pass:**
    *   Query Stuff MLP with $(\mathbf{x}, \mathbf{d})$ to get $\sigma_s, \mathbf{c}_s, \mathbf{l}_s$ [S2].
    *   For each intersecting Thing $k$, query MLP with $(\mathbf{x}', \mathbf{d})$ to get $\sigma_k, \mathbf{c}_k, \mathbf{l}_k$ [S2].
    *   Compose fields using the aggregation formula [S2].
    *   Perform volumetric rendering to synthesize color, semantic, and depth images [S2].
4.  **Loss Calculation:**
    *   **Color Loss:** Mean Squared Error (MSE) between rendered and input RGB images [S2].
    *   **Semantic Loss:** Softmax cross-entropy between rendered semantic logits and predicted 2D semantic labels [S2].
    *   **Total Loss:** $\mathcal{L} = \mathcal{L}_{color} + \lambda \mathcal{L}_{semantic}$, where $\lambda < 1$ scales semantic loss to prevent dominance [S2].
5.  **Optimization:**
    *   Use Adam optimizer [S2].
    *   Update MLP weights and bounding box parameters ($\mathbf{R}, \mathbf{t}, \mathbf{s}$) [S2].
    *   **Rotation Orthogonalization:** After each gradient step, orthogonalize $\mathbf{R}$ using SVD to project back onto SO(3) [S2].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def positional_encoding(p, L):
    """
    Hierarchical positional encoding for scalar p.
    p: scalar or array of scalars
    L: number of frequency bands
    Returns: encoded vector
    """
    # p is expected to be in range [0, 1] or similar, scaled appropriately
    # Formula: (sin(2^0 pi p), cos(2^0 pi p), ..., sin(2^{L-1} pi p), cos(2^{L-1} pi p))
    scales = np.array([2**i for i in range(L)])
    encoded = []
    for scale in scales:
        encoded.append(np.sin(np.pi * scale * p))
        encoded.append(np.cos(np.pi * scale * p))
    return np.concatenate(encoded)

def encode_vector(v, L):
    """
    Encode each component of vector v independently.
    v: array of shape (3,)
    L: number of frequencies
    Returns: encoded array
    """
    enc_x = positional_encoding(v[0], L)
    enc_y = positional_encoding(v[1], L)
    enc_z = positional_encoding(v[2], L)
    return np.concatenate([enc_x, enc_y, enc_z])

def orthogonalize_R(R):
    """
    Orthogonalize rotation matrix R using SVD.
    R: 3x3 matrix
    Returns: orthogonalized 3x3 matrix
    """
    U, _, Vt = np.linalg.svd(R)
    return U @ Vt

def query_stuff_mlp(x_world, d, stuff_params):
    """
    Query the stuff MLP.
    x_world: 3D position
    d: view direction
    stuff_params: MLP weights
    Returns: sigma_s, c_s, l_s
    """
    # Encode position and direction
    # Stuff uses L=10 for position, L=2 for direction
    enc_pos = encode_vector(x_world, L=10)
    enc_dir = encode_vector(d, L=2)
    
    # Concatenate and pass through MLP (8 layers, width 256)
    # Note: Density and semantic do not depend on direction, color does.
    # Simplified MLP forward pass
    x = np.concatenate([enc_pos, enc_dir])
    for layer in stuff_params['layers']:
        x = np.maximum(0, x @ layer['W'] + layer['b']) # ReLU
    
    # Split outputs based on architecture
    # Assuming last layers split into sigma, c, l
    # This is a placeholder for the actual MLP structure defined in [S2]
    sigma_s = x[0]
    c_s = x[1:4]
    l_s = x[4:]
    return sigma_s, c_s, l_s

def query_thing_mlp(x_local, d, thing_params):
    """
    Query the thing MLP for instance k.
    x_local: 3D position in object local coordinates
    d: view direction
    thing_params: MLP weights for instance k
    Returns: sigma_k, c_k, l_k
    """
    # Encode position and direction
    # Thing uses L=6 for position, L=2 for direction
    enc_pos = encode_vector(x_local, L=6)
    enc_dir = encode_vector(d, L=2)
    
    x = np.concatenate([enc_pos, enc_dir])
    for layer in thing_params['layers']:
        x = np.maximum(0, x @ layer['W'] + layer['b']) # ReLU
        
    sigma_k = x[0]
    c_k = x[1:4]
    l_k = x[4:]
    return sigma_k, c_k, l_k

def render_ray(ray_origin, ray_direction, time, stuff_params, things, things_tracks):
    """
    Render a single ray.
    ray_origin: 3D origin
    ray_direction: 3D direction
    time: float timestamp
    stuff_params: Stuff MLP parameters
    things: List of Thing MLP parameters
    things_tracks: List of object track parameters (R, t, s)
    Returns: rendered color, density, semantic, instance
    """
    # Sample points along ray (1024 samples, stratified)
    # Assume samples are generated as lambda values
    lambdas = np.linspace(0, 1, 1024) # Placeholder for actual near/far range
    points_world = ray_origin + lambdas[:, None] * ray_direction
    
    # Initialize accumulators
    accumulated_alpha = 0.0
    accumulated_color = np.zeros(3)
    accumulated_density = 0.0
    accumulated_semantic = np.zeros(num_classes) # Placeholder
    accumulated_instance = np.zeros(num_instances) # Placeholder
    
    for i, x_world in enumerate(points_world):
        # Check intersection with bounding boxes
        # Determine which things intersect x_world at time
        intersecting_things = []
        for k, track in enumerate(things_tracks):
            # Interpolate track to get T_k at time
            T_k = interpolate_track(track, time)
            # Transform world point to local coordinates
            x_local = transform_to_local(x_world, T_k)
            # Check if x_local is inside bounding box s
            if is_inside_box(x_local, track['s']):
                intersecting_things.append(k)
        
        # Compute field value at this point
        # Default to stuff
        sigma_s, c_s, l_s = query_stuff_mlp(x_world, ray_direction, stuff_params)
        
        # Add contributions from intersecting things
        sigma_total = sigma_s
        c_total = c_s
        l_total = l_s
        instance_id = 0 # Stuff instance ID
        
        for k in intersecting_things:
            sigma_k, c_k, l_k = query_thing_mlp(x_world, ray_direction, things[k]) # Note: x_world transformed inside query or here? Formula uses T_k^-1 x
            # Actually, query_thing_mlp expects local coords. Let's adjust.
            # Re-query with local coords
            sigma_k, c_k, l_k = query_thing_mlp(x_local, ray_direction, things[k])
            
            # Aggregate
            sigma_total += sigma_k
            c_total += c_k
            l_total += l_k
            instance_id = k + 1 # Thing instance IDs
            
        # Volume rendering step (simplified)
        # alpha = 1 - exp(-sigma * delta)
        # weight = alpha * exp(-sum(sigma * delta))
        # accumulated_color += weight * c
        # ...
        pass
        
    return accumulated_color, accumulated_density, accumulated_semantic, accumulated_instance

def train_step(batch, stuff_params, things, things_tracks):
    """
    Perform one training step.
    """
    # Forward pass to render images
    rendered_colors, rendered_semantics = render_batch(batch, stuff_params, things, things_tracks)
    
    # Compute losses
    color_loss = mse(rendered_colors, batch['rgb'])
    semantic_loss = cross_entropy(rendered_semantics, batch['semantic'])
    total_loss = color_loss + lambda_sem * semantic_loss
    
    # Backward pass
    gradients = compute_gradients(total_loss, stuff_params, things, things_tracks)
    
    # Update parameters
    stuff_params = adam_update(stuff_params, gradients['stuff'])
    things = adam_update(things, gradients['things'])
    things_tracks = adam_update(things_tracks, gradients['tracks'])
    
    # Orthogonalize rotations
    for k in range(len(things_tracks)):
        things_tracks[k]['R'] = orthogonalize_R(things_tracks[k]['R'])
        
    return total_loss
```

## Subtopics

### Panoptic Radiance Field
Uses shared MLPs with latent codes for objects, limiting instance-specific detail reproduction compared to PNF's separate MLPs [S2].

### Category-Level Semantic NeRF
Lacks object awareness and dynamic scene handling, unlike PNF which explicitly decomposes things and stuff [S2].

## Operating characteristics
*   **Time and Memory Complexity:** Compute-intensive, limited to offline applications [S2]. Uses 1024 samples per ray [S2].
*   **Measured Performance:** Outperforms SemanticNeRF by +9.2 mIoU on dynamic scenes (KITTI) [S2]. Uses 10x fewer FLOPs in instance MLPs compared to NSG due to meta-learning priors [S2].
*   **Strengths:** Handles dynamic scenes, object-aware, enables scene editing (removing/adding objects) [S2].
*   **Limitations:** Does not handle missing/duplicate detections, incorrect class predictions, deformable objects, or complex light transport (e.g., shadowing) [S2].

## Background
Panoptic Neural Fields (PNF) was introduced by Google Research, Georgia Tech, Simon Fraser University, and Stanford University in May 2022 [S2]. It builds on NeRF [S2] and addresses limitations of previous object-aware frameworks that shared MLPs or lacked dynamic handling [S2]. It is evaluated on KITTI and KITTI-360 datasets [S2].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] Panoptic Neural Fields: A Semantic Object-Aware Neural Scene Representation, https://doi.org/10.1109/cvpr52688.2022.01253, fetched 2026-10-26
[S3] Semantically-aware Neural Radiance Fields for Visual Scene Understanding: A Comprehensive Review, https://link.springer.com/article/10.1007/s11263-025-02663-5, fetched 2026-10-27
