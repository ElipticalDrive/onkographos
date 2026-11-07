# Pulsar Sphere-based Neural Rendering

Also called: Pulsar

## Overview
Pulsar is an efficient, sphere-based differentiable renderer that maps a 3D scene representation consisting of millions of spheres to a rendered feature image using camera parameters (rotation $R$, translation $t$, intrinsic $K$) [S1]. It solves the problem of real-time rendering and optimization for large-scale point-based radiance fields by replacing traditional acceleration structures (like BVHs or k-d-trees) with parallelized bounding box projections and sorting [S1]. Unlike its parent method, Differentiable Point-Based Radiance Fields, which may rely on general splatting, Pulsar specifically optimizes the projection step to handle millions of spheres with low latency, enabling real-time performance [S1].

## How it works
**Inputs:** A set of $M$ spheres $S = \{(p_i, f_i, r_i, o_i)\}_{i=1}^M$, where $p_i \in \mathbb{R}^3$ is the position, $f_i \in \mathbb{R}^d$ is the neural feature vector, $r_i \in \mathbb{R}$ is the radius, and $o_i \in \mathbb{R}$ is the opacity [S1]. Camera parameters: rotation $R$, translation $t$, and intrinsic matrix $K$ [S1].

**Outputs:** A rendered feature image $F$ [S1].

**Data Structures:**
The renderer converts the scene from a "struct of arrays" to an "array of structs" format using two primary data structures [S1]:

1.  **Intersection Information Struct:**
    *   **Fields:** Four `unsigned short` values representing the minimum and maximum x and y limits of the sphere's projection on the sensor plane [S1].
    *   **Size:** 8 bytes [S1].
    *   **Layout:** Fits exactly into one cache line on NVIDIA Turing GPUs (256 bytes / 32 threads = 8 bytes per thread), enabling coalesced loading [S1].
    *   **Auxiliary:** An additional array stores the earliest possible intersection depth for each sphere (set to infinity if outside the sensor) [S1].

2.  **Draw Information Struct:**
    *   **Fields:** Position vector (3 floats), up to three feature value floats or a float pointer (if $d > 3$), distance to sphere center, and sphere radius [S1].
    *   **Size:** 32 bytes [S1].

**Parameters:**
*   $\gamma$: Temperature parameter for the exponential blending function [S1].
*   $z_i$: Normalized device coordinate (NDC) for depth, where $0$ denotes maximum depth and $1$ denotes minimum depth (closest to camera) [S1].
*   Block Size: 16x16 = 256 threads (8 warps per block) [S1].

**Algorithm Steps:**

1.  **Step 0: Enclosing Rectangle Calculation and Sorting**
    *   Parallelize over all spheres using the full GPU [S1].
    *   For each sphere $i$, calculate the bounding box projection on the sensor plane using $K$, $p_i$, and $r_i$ [S1].
    *   Populate the Intersection Information and Draw Information structs [S1].
    *   Calculate the earliest possible intersection depth for each sphere [S1].
    *   Sort the Intersection Information and Draw Information arrays by earliest intersection depth using the CUB library [S1].
    *   *Note:* This step takes 0.22 ms for 1,000,000 spheres for computation/storage and 3.2 ms for sorting [S1].

2.  **Step 1: Intersection Search**
    *   Divide the image into nine sequential parts (an empirically derived value) [S1].
    *   For each part:
        *   Use the full GPU to iterate over the sorted Intersection Information array to identify spheres relevant to the current image part [S1].
        *   Use CUB `select_flags` to create sorted subsets of Intersection Information and Draw Information for the relevant spheres [S1].
        *   Launch thread blocks of size 16x16 (256 threads) to process pixels in the current part [S1].
        *   Use coalesced processing via warps to test sphere limits against pixel coordinates [S1].
        *   Store sphere draw information for intersecting spheres in block-wide shared memory [S1].

3.  **Step 2: Blending and Rendering**
    *   For each pixel, iterate over the relevant spheres (sorted by depth) [S1].
    *   Calculate the blending weight $w_i$ for each sphere $i$ using the commutative blending function [S1]:
        $$w_i = \frac{o_i \cdot d_i \cdot \exp(o_i \cdot \frac{z_i}{\gamma})}{\exp(\frac{\xi}{\gamma}) + \sum_k o_k \cdot d_k \cdot \exp(o_k \cdot \frac{z_k}{\gamma})}$$
        where $\xi$ is the denominator term excluding sphere $i$'s contribution (or the sum over all $k$ in the denominator context) [S1].
    *   Calculate the normalized orthogonal distance $d_i$:
        $$d_i = \min(1, \frac{||\tilde{d}_i||_2}{R_i})$$
        where $\tilde{d}_i$ is the vector pointing orthogonally from the ray to the sphere center, and $R_i$ is the sphere radius [S1].
    *   Accumulate features weighted by $w_i$ to produce the final pixel color/feature [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def pulsar_render(spheres, camera_params, image_shape, gamma):
    """
    spheres: Tensor of shape (M, 3 + d + 2)
             Columns: [px, py, pz, f_1...f_d, r, o]
             Note: Features f are assumed to be packed or handled via pointers in C++/CUDA.
             For this Python-like pseudocode, we assume d <= 3 for simplicity or handle pointer logic abstractly.
    camera_params: Dict with 'K', 'R', 't'
    image_shape: Tuple (H, W)
    gamma: Float, temperature parameter
    """
    M = spheres.shape[0]
    K = camera_params['K']
    R = camera_params['R']
    t = camera_params['t']
    H, W = image_shape
    
    # Step 0: Enclosing Rectangle Calculation
    # Transform sphere centers to camera space
    # p_cam = R * p_world + t
    p_world = spheres[:, :3]
    p_cam = torch.matmul(R, p_world.t()).t() + t
    
    # Project to normalized device coordinates or screen space
    # Using pinhole model approximation for bounding box
    # x = K[0,0] * X/Z + K[0,2]
    # y = K[1,1] * Y/Z + K[1,2]
    # We need the bounding box of the sphere in screen space.
    # This involves trigonometric identities to avoid distortion issues [S1].
    # For pseudocode, we assume a helper function `get_screen_bbox` exists
    # which returns min_x, max_x, min_y, max_y, and depth_z for each sphere.
    
    bbox_data = get_screen_bbox(p_cam, spheres[:, -2], K) # Returns (M, 4) for x,y limits and depth info
    
    # Store earliest intersection depth (z)
    # Spheres outside sensor get infinity
    depths = bbox_data[:, 4] # Assuming 5th column is depth
    valid_mask = (depths > 0) & (depths < float('inf'))
    
    # Sort spheres by depth (earliest intersection first)
    # Using CUB-like sort in practice
    sorted_indices = torch.argsort(depths)
    
    sorted_bbox = bbox_data[sorted_indices]
    sorted_spheres = spheres[sorted_indices]
    
    # Step 1: Intersection Search
    # Divide image into 9 parts
    part_h, part_w = H // 3, W // 3
    parts = []
    for i in range(3):
        for j in range(3):
            y_min, y_max = i * part_h, (i + 1) * part_h
            x_min, x_max = j * part_w, (j + 1) * part_w
            parts.append((y_min, y_max, x_min, x_max))
            
    # Initialize output image
    # Assuming d=3 for feature channels (RGB) for this example
    d = sorted_spheres.shape[1] - 5 # 3 pos + 1 radius + 1 opacity = 5
    if d < 3:
        d = 3 # Fallback for pseudocode
        
    features = sorted_spheres[:, 3:3+d]
    opacities = sorted_spheres[:, -1]
    radii = sorted_spheres[:, -2]
    
    # Pre-allocate result
    # In CUDA, this is done per block/pixel. Here we simulate the accumulation.
    # We need to iterate over pixels and then spheres.
    
    # To make this self-contained and valid Python without external libs for the core logic:
    # We will compute weights for a single ray/pixel to demonstrate the formula,
    # then aggregate.
    
    # Note: The actual Pulsar implementation uses coalesced warps and shared memory.
    # This Python code demonstrates the mathematical accumulation for a single pixel.
    
    def compute_pixel_weight(y, x, spheres_subset, gamma):
        """
        Computes the blended feature for a single pixel (y, x) given a subset of spheres.
        """
        # Get spheres relevant to this pixel based on bbox
        # In CUDA, this is done via the Intersection Info struct check.
        # Here we filter manually for demonstration.
        relevant_spheres = []
        for idx, sphere in enumerate(spheres_subset):
            # Check if pixel (x,y) is within sphere's bbox
            # bbox_data was sorted along with spheres
            # We need to access the bbox for this specific sphere
            # This is inefficient in Python but correct for logic demonstration
            # In CUDA, this is a simple struct access.
            pass 
            
        # Simplified: Assume we have a list of (sphere, bbox) tuples for this pixel
        # Calculate weights
        weights = []
        for sphere, bbox in relevant_spheres:
            p_cam = sphere[:3]
            r = sphere[-2]
            o = sphere[-1]
            
            # Calculate z_i (NDC depth)
            # z_i in [0, 1], 0=max depth, 1=min depth
            # Assuming standard NDC mapping or specific Pulsar mapping
            # Pulsar uses normalized device coordinates where 0 is max depth
            z_i = 1.0 - (p_cam[2] / max_depth) # Placeholder for exact NDC mapping
            
            # Calculate d_i (normalized orthogonal distance)
            # Vector from ray to sphere center
            # Ray direction is roughly (0,0,1) in camera space for central pixels,
            # but generally depends on camera intrinsics.
            # For a pixel (x,y), the ray direction v can be computed from K.
            # d_tilde is orthogonal component.
            
            # Simplified d_i calculation for pseudocode:
            # d_tilde = p_cam - projection_of_p_cam_on_ray
            # d_i = min(1, ||d_tilde|| / r)
            
            # Placeholder for orthogonal distance calculation
            d_i = calculate_orthogonal_distance(p_cam, x, y, K)
            d_i = min(1.0, d_i / r)
            
            weights.append((o, d_i, z_i))
            
        # Compute blending weights
        # w_i = (o_i * d_i * exp(o_i * z_i / gamma)) / (exp(xi/gamma) + sum(o_k * d_k * exp(o_k * z_k / gamma)))
        
        numerator_terms = []
        denominator_sum = 0.0
        
        for o, d, z in weights:
            term = o * d * math.exp(o * z / gamma)
            numerator_terms.append(term)
            denominator_sum += term
            
        # xi is the denominator term excluding sphere i's contribution?
        # The formula says: exp(xi/gamma) + sum(...)
        # Actually, the denominator is: exp(xi/gamma) + sum_k (o_k * d_k * exp(o_k * z_k / gamma))
        # Wait, the source says: exp(xi/gamma) + sum_i ...
        # And xi is "denominator term excluding sphere i's contribution"
        # This implies the denominator is dynamic per sphere?
        # "The function is commutative with respect to sphere order."
        # If it's commutative, the order doesn't matter.
        # Let's look at the formula again:
        # w_i = (numerator_i) / (exp(xi/gamma) + sum_all)
        # If xi excludes i, then the denominator is exp(xi/gamma) + numerator_i + sum_others?
        # No, the source says: "xi is the denominator term excluding sphere i's contribution (or sum over all k in the denominator)"
        # This phrasing is ambiguous. Let's look at Eq 2 in source:
        # w_i = ... / (exp(xi/gamma) + sum_i o_k d_k exp(...))
        # If xi excludes i, then the sum is over all k.
        # This would mean the denominator depends on i.
        # However, standard alpha blending is commutative.
        # Pulsar's blending is commutative [S1].
        # If it is commutative, the denominator must be independent of the order or the specific i in a way that preserves commutativity.
        # Let's assume the standard interpretation:
        # Denominator = sum_k (o_k * d_k * exp(o_k * z_k / gamma)) + exp(xi/gamma)
        # If xi is a constant or global term, it's easier.
        # But the source says "xi is the denominator term excluding sphere i's contribution".
        # This suggests: Denom_i = exp( (Sum_{k!=i} term_k) / gamma ) + term_i ?
        # No, that's not commutative.
        # Let's re-read carefully: "xi is the denominator term excluding sphere i's contribution (or sum over all k in the denominator)"
        # This might mean xi = sum_{k!=i} (o_k d_k exp(...)).
        # Then Denom = exp(xi/gamma) + sum_all?
        # This seems complex.
        # Given the constraint "commutative", let's look at the structure.
        # If w_i is commutative, the set of weights must sum to 1 regardless of order.
        # The formula provided is:
        # w_i = N_i / (exp(xi/gamma) + S)
        # If xi excludes i, then S includes i?
        # Let's assume the source implies a specific normalization.
        # For the purpose of implementation, we calculate the sum of all terms S_total.
        # If the formula is strictly as written, we must compute xi for each i.
        
        # Implementation of the specific Pulsar formula:
        total_sum = sum(numerator_terms)
        
        final_weights = []
        for idx, (o, d, z) in enumerate(weights):
            term = numerator_terms[idx]
            # xi is sum of all other terms
            xi = total_sum - term
            denom = math.exp(xi / gamma) + total_sum
            w = term / denom
            final_weights.append(w)
            
        return final_weights, features

    # This Python code is illustrative of the math. The actual CUDA kernel handles the parallel reduction and sorting.
    pass

def calculate_orthogonal_distance(p_cam, x, y, K):
    # Placeholder for the orthogonal distance calculation
    # Returns ||d_tilde||
    return 0.0

def get_screen_bbox(p_cam, r, K):
    # Placeholder for bounding box calculation
    # Returns [min_x, max_x, min_y, max_y, depth_z]
    return torch.zeros((p_cam.shape[0], 5))
```

**Edge Cases and Pitfalls:**
*   **Position Gradient Instability:** Position gradients are numerically unstable if a sphere projects to a radius of 3 pixels or fewer [S1]. To handle this, position gradients are only provided for spheres with a projected radius greater than 3 pixels [S1].
*   **Lost Spheres:** To prevent spheres from being "lost" between pixel rays and ensure every sphere receives gradients, every sphere is rendered with at least a size of one pixel [S1]. This robustifies calculated boundaries against numerical inaccuracies [S1].

## Subtopics

### Sphere Subdivision Refinement
A technique to increase sphere count and detail by subdividing existing spheres into smaller ones, followed by re-optimization and pruning [S1]. It refines each sphere with 12 new spheres with radius $\sqrt{2} \cdot r$ (where $r$ is the previous radius), placed in a face-centered cubic packing scheme relative to the parent sphere [S1]. After subdivision, the properties of the new spheres are optimized, and pruning is applied to eliminate spheres whose color converges towards the background or which are not visible (do not receive gradient updates) [S1].

### Neural Shading in Pulsar
A method to model view-dependent appearance by adding a neural shading network that conditions on per-pixel view direction, shared across all pixels [S1]. It uses a fully-connected neural network (e.g., Pix2PixHD design with 15 feature channels) conditioned on per-pixel view direction [S1]. Gradients are back-propagated through the shading model to optimize channel information (sphere colors/features) using photometric L1-loss, perceptual loss, and adversarial loss [S1].

## Operating characteristics
*   **Time Complexity:** The projection step (Step 0) takes 0.22 ms for 1,000,000 spheres for computation/storage and 3.2 ms for sorting [S1]. The renderer enables real-time rendering and optimization of representations with millions of spheres [S1].
*   **Memory Complexity:** The data structures are compact: 8 bytes for Intersection Information and 32 bytes for Draw Information per sphere [S1].
*   **Performance:** Pulsar executes orders of magnitude faster than existing techniques [S1]. In a 3D reconstruction experiment, Pulsar completed optimization in 31 seconds compared to 20 minutes for DSS [S1].
*   **Strengths:** High speed, scalability to millions of spheres, real-time rendering, and easy integration with PyTorch [S1].
*   **Limitations:** Relies on specific GPU architectures (NVIDIA Turing+) for optimal cache line alignment and warp efficiency [S1]. The block size is limited by memory requirements (256 threads) on current hardware, though it may scale with future GPUs [S1].
*   **When to use:** When real-time rendering and optimization of large-scale sphere-based neural representations are required [S1].
*   **When not to use:** When using hardware that does not support the specific warp/cache optimizations or when acceleration structures like BVHs are preferred for non-spherical primitives [S1].

## Background
Pulsar was presented in 2021 by the authors of the paper "Pulsar: Efficient Sphere-based Neural Rendering" [S1]. It builds on recent insights in differentiable rendering and neural networks [S1]. The method was developed to address the computational bottlenecks of existing differentiable point-based renderers, specifically by limiting complexity in the projection step [S1]. It is implemented as open-source software with tight integration with PyTorch [S1].

## Sources
[S1] Pulsar: Efficient Sphere-based Neural Rendering, https://doi.org/10.1109/cvpr46437.2021.00149, fetched 2026-10-26
