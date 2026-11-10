# Adaptive Ray Shooting & Quadtree Subdivision

Also called: Adaptive quadtree subdivision, Image context based probability distribution

## Overview
Adaptive Ray Shooting & Quadtree Subdivision is a general strategy to accelerate the training of radiance fields by reducing ray redundancy during volume rendering [S1]. It solves the problem of excessive computational cost in multi-view volume rendering by dynamically adjusting sampling density based on image complexity [S1]. Unlike methods requiring specific architectures like sparse voxel grids or hash coding, this approach works with almost any radiance field method by subdividing views into a quadtree and shooting fewer rays in simple regions while maintaining high density in complex areas [S1].

## How it works
The method operates on the standard 5D point representation used in neural radiance fields, where a 3D spatial coordinate $\mathbf{x} = (x, y, z) \in \mathbb{R}^3$ and a 2D viewing direction $\mathbf{d} = (\theta, \phi) \in \mathbb{R}^2$ are mapped to emitted color $\mathbf{c}$ and volume density $\sigma$ [S1]. The core mechanism involves two components: a probability-based sampling function driven by image context and an adaptive quadtree subdivision strategy [S1].

**Inputs and Outputs**
*   **Inputs:** A set of $N$ training images $\mathcal{I}$; a neural radiance field function $V_\Theta$ that maps a ray to rendered color; target pixel colors $c_{ki}$.
*   **Outputs:** A set of rays $R_i$ for each image $I_i$, with density adapted to local error.

**Data Structures**
*   **Quadtree:** Each view is subdivided into a quadtree. Each node stores the average rendering error of the rays passing through its corresponding image region [S1].
*   **Ray Set:** $R_i$ is the set of rays generated from image $I_i$. $r_{ki}$ denotes the $k$-th ray from the $i$-th image [S1].

**Parameters**
*   $N$: Number of training images [S1].
*   $M$: Number of rays per image (base count) [S1].
*   Threshold: Implicit threshold for quadtree subdivision based on average rendering error [S1].

**Algorithm Steps**
1.  **Initial Ray Generation:** Generate an initial set of rays $R_i$ for each training image $I_i \in \mathcal{I}$ [S1].
2.  **Rendering and Error Calculation:** Render the scene using the current radiance field $V_\Theta$ to obtain predicted colors $\hat{c}_{ki} = V_\Theta(r_{ki})$ [S1]. Compute the volume rendering loss $L$ using the $L_2$ norm between predicted and target colors:
    $$ L = \frac{1}{NM} \sum_{I_i \in \mathcal{I}} \sum_{r_{ki} \in R_i} ||V_\Theta(r_{ki}) - c_{ki}||_2^2 $$
    where $c_{ki}$ is the target pixel color for ray $r_{ki}$ [S1].
3.  **Quadtree Subdivision:** Subdivide each view into a quadtree structure. Calculate the average rendering error for each node in the quadtree based on the rays passing through that region [S1].
4.  **Adaptive Refinement:** Identify regions with large rendering error (complex regions) and regions with small error (simple regions) [S1]. Dynamically shoot more rays in complex regions and fewer rays in simple regions [S1]. Specifically, prioritize shooting rays at pixels with dramatic color change [S1].
5.  **Training:** Optimize the network parameters by minimizing the loss $L$ using the adaptively sampled rays [S1].
6.  **Iteration:** Repeat steps 2-5 until convergence [S1].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_ray_shooting_training(training_images, initial_ray_count, radiance_field, target_colors):
    """
    Accelerates radiance field training using adaptive ray shooting and quadtree subdivision.
    
    Args:
        training_images: List of training images I_i.
        initial_ray_count: Base number of rays M per image.
        radiance_field: Function V_Theta mapping rays to rendered colors.
        target_colors: Ground truth pixel colors c_ki.
        
    Returns:
        Optimized radiance field parameters.
    """
    num_images = len(training_images)
    rays_per_image = initial_ray_count
    
    # Initialize quadtree structures for each image view
    quadtrees = [create_quadtree(image.width, image.height) for image in training_images]
    
    for epoch in range(max_epochs):
        total_loss = 0.0
        all_sampled_rays = []
        
        for i, image in enumerate(training_images):
            # Step 1: Generate rays based on current quadtree subdivision
            # Simple regions have fewer rays, complex regions have more
            current_rays = generate_adaptive_rays(image, quadtrees[i], rays_per_image)
            
            # Step 2: Render using the radiance field
            rendered_colors = []
            for ray in current_rays:
                # V_Theta maps ray to color c_hat
                pred_color = radiance_field(ray)
                rendered_colors.append(pred_color)
            
            # Step 3: Calculate loss for this image
            # L_i = (1/M) * sum(||V_Theta(r_ki) - c_ki||_2^2)
            image_loss = 0.0
            for k, ray in enumerate(current_rays):
                target = target_colors[i][k] # Corresponding target color for ray k
                error = squared_l2_norm(rendered_colors[k] - target)
                image_loss += error
            
            image_loss /= len(current_rays)
            total_loss += image_loss
            
            # Step 4: Update Quadtree based on error distribution
            # Subdivide nodes with high average error
            update_quadtree(quadtrees[i], current_rays, rendered_colors, target_colors[i])
            
            all_sampled_rays.extend(current_rays)
            
        # Step 5: Optimize radiance field parameters to minimize total loss
        # L = (1 / (N * M)) * sum over all images and rays of ||V_Theta(r_ki) - c_ki||_2^2
        optimize_radiance_field(radiance_field, all_sampled_rays, target_colors, total_loss)
        
    return radiance_field

def update_quadtree(quadtree, rays, rendered_colors, targets):
    """
    Updates the quadtree nodes with average rendering error to guide next iteration's sampling.
    """
    for node in quadtree.nodes:
        # Identify rays passing through this node's region
        node_rays = [r for r in rays if r.passes_through(node.region)]
        
        if len(node_rays) > 0:
            # Calculate average error for this node
            node_error_sum = 0.0
            for ray in node_rays:
                # Find corresponding rendered color and target
                # (Assuming ray index maps to color/target index)
                idx = rays.index(ray)
                error = squared_l2_norm(rendered_colors[idx] - targets[idx])
                node_error_sum += error
            
            node.average_error = node_error_sum / len(node_rays)
            
            # Subdivision logic: if error > threshold, mark for subdivision
            if node.average_error > error_threshold:
                node.mark_for_subdivision()
```

**Edge Cases and Pitfalls**
*   **Precision:** The loss calculation relies on the $L_2$ norm, which is sensitive to large outliers in color error [S1].
*   **Boundaries:** The quadtree subdivision must handle image boundaries correctly to ensure all pixels are covered [S1].
*   **Degenerate Input:** If the radiance field is not yet trained, initial error estimates may be noisy, leading to poor subdivision decisions [G].

## Operating characteristics
*   **Time Complexity:** Reduces training time by significantly lowering the number of rays shot during volume rendering [S1]. The exact complexity depends on the underlying radiance field architecture, but the method itself adds overhead for quadtree management and error calculation [S1].
*   **Memory Complexity:** Requires additional memory to store the quadtree structures for each view and track ray errors [S1].
*   **Measured Performance:** Evaluated with NeRF and Plenoxels under widely used benchmarks, achieving comparable accuracy to state-of-the-art methods with much faster training [S1].
*   **Strengths:** General strategy that works for almost all radiance field-based methods without requiring specific architectural modifications [S1]. Reduces training burden by focusing rays on pixels with dramatic color changes [S1].
*   **Limitations:** Accuracy may suffer if the error metric does not correlate well with perceptual quality or if the quadtree subdivision is too coarse [G].
*   **When to use:** When training mainstream radiance field methods and computational efficiency is a priority [S1].
*   **When not to use:** When specific architectural optimizations (like Instant-NGP's hash coding) are already in place and provide sufficient speed [S1].

## Background
The method was introduced to address the limitations of specialized approaches like Plenoxels, TensoRF, and Instant-NGP, which require specific architectures such as sparse voxel grids, discrete tensor coordinates, or hash coding [S1]. These specialized methods are not easily adaptable to improve the training efficiency of different neural radiance field variations [S1]. The authors proposed this general strategy to speed up the learning procedure for almost all radiance field-based methods by reducing ray redundancy in the multi-view volume rendering procedure [S1]. It was evaluated with NeRF and Plenoxels under widely used benchmarks [S1].

## Sources
[S1] Fast Learning Radiance Fields by Shooting Much Fewer Rays, https://doi.org/10.1109/tip.2023.3267049, fetched 2026-10-26
