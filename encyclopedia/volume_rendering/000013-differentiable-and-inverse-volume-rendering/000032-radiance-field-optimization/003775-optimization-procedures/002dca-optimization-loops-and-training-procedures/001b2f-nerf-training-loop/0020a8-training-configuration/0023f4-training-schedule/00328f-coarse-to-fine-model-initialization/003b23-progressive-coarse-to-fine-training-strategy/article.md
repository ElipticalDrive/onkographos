# Progressive Coarse-to-Fine Training Strategy

Also called: Training Strategy

## Overview
This method accelerates radiance-field optimization by decoupling structural learning from high-frequency detail recovery. It solves the problem of slow convergence and instability in dynamic scenes by first establishing global geometry at low resolution, then refining appearance and directionality at high resolution. It sets itself apart from single-resolution training by progressively adding temporal data and switching feature representations, building on [Coarse-to-Fine Model Initialization](6793-coarse-to-fine-model-initialization.md).

## How it works
**Inputs and Outputs:**
- **Inputs:** A set of dynamic scene images with associated camera poses and time steps.
- **Outputs:** A trained radiance field model (final model) capable of novel view synthesis.

**Data Structures:**
- **Proxy Geometry:** A coarse representation used to compute a tighter bounding box.
- **Color Features:**
  - Coarse Stage: Scalar color values (no ray direction dependency).
  - Fine Stage: High-dimensional feature vectors (ray direction dependency modeled via $F_{\theta4}$).

**Numbered Steps:**
1. **Progressive Data Addition:** Begin training with images close to the canonical time step. Iteratively add images with increasingly distant time steps until the full dataset is included [S1].
2. **Coarse Stage Initialization:**
   - Set color features to be the color itself, ignoring ray directions [S1].
   - Compute a smaller bounding box using proxy geometry learned during this stage to filter empty space [S1].
   - Incrementally insert new observations into the training set at fixed intervals [S2, unofficial].
3. **Coarse Stage Optimization:**
   - Sample rays from input images.
   - Train the coarse model for a specified number of iterations [S2, unofficial].
4. **Fine Stage Transition:**
   - Switch color features to high-dimensional vectors.
   - Model ray direction dependency using a dedicated network component $F_{\theta4}$ [S1].
   - Use the tighter bounding box derived from the coarse stage to restrict the volume of interest [S1].
5. **Fine Stage Optimization:**
   - Train the final model using all observations (no incremental insertion) [S2, unofficial].
   - Perform refinement for the specified number of iterations [S2, unofficial].

**Formulas:**
The total loss function includes terms for photometric error, point cloud consistency, background consistency, flow consistency, and total variation (TV) regularization on density, flow, and displacement fields [S1]:
$$
\mathcal{L} = w_1\mathcal{L}_{ptc} + w_2\mathcal{L}_{bg} + w_3\mathcal{L}_{flow} + w_4\mathcal{L}_{vdiff} + w_5\mathcal{L}_{tv}(\mathbf{V}_{\sigma}^{\text{Can}}) + w_6\mathcal{L}_{tv}(\mathbf{V}_{\text{flow}}^t) + w_7\mathcal{L}_{tv}(\mathbf{D})
$$
Where $w_i$ are weighting factors [S1].

**Parameters:**
- **Ray Sample Count:** 8,192 rays per optimization step [S2, unofficial].
- **Coarse Training Iterations:** 5,000 [S2, unofficial].
- **Refinement Iterations:** 20,000 [S2, unofficial].
- **Observation Insertion Interval:** Every 50 iterations during the coarse stage [S2, unofficial].
- **Optimizer:** Adam [S2, unofficial].
- **Initial Learning Rate:** $10^{-1}$ [S2, unofficial].
- **Scene Parameters:** Grid size and other scene-specific parameters are dataset-dependent, adopted from [59] (Not given in the sources).

**Edge Cases and Pitfalls:**
- **Precision:** The transition from scalar color to high-dimensional features requires careful initialization to avoid instability [S1].
- **Boundaries:** The proxy geometry must accurately estimate the scene extent to ensure the smaller bounding box does not clip valid geometry [S1].
- **Degenerate Input:** If the canonical time step is poorly chosen, progressive addition may lead to suboptimal global geometry [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def progressive_coarse_fine_training(scene_data, canonical_time_idx):
    # Initialize model with scalar color features
    model = RadianceField(color_feature_type="scalar")
    
    # Sort images by temporal distance from canonical time
    sorted_images = sort_by_temporal_distance(scene_data, canonical_time_idx)
    
    # Progressive data addition
    current_images = []
    for i, image in enumerate(sorted_images):
        current_images.append(image)
        
        # Coarse Stage
        if i < len(sorted_images) - 1: # Not all images added yet
            # Incremental insertion: every 50 iterations
            if i % 50 == 0:
                insert_new_observations(model, current_images)
            
            # Sample rays
            rays = sample_rays(current_images, num_rays=8192)
            
            # Train coarse model
            for _ in range(5000):
                loss = compute_loss(model, rays)
                optimize(model, loss, optimizer="Adam", lr=1e-1)
            
            # Update proxy geometry for bounding box
            proxy_geometry = model.get_proxy_geometry()
            bounding_box = compute_bounding_box(proxy_geometry)
        else:
            # Fine Stage
            # Switch to high-dimensional features
            model.switch_to_high_dim_features()
            
            # Use tighter bounding box
            model.set_bounding_box(bounding_box)
            
            # Train with all images
            all_rays = sample_rays(scene_data, num_rays=8192)
            for _ in range(20000):
                loss = compute_loss(model, all_rays)
                optimize(model, loss, optimizer="Adam", lr=1e-1)
                
    return model
```

## Subtopics

### Incremental Mapping Optimization
This variant focuses on the coarse stage's incremental data insertion. It samples 8,192 rays per step, trains for 5,000 iterations, and inserts new observations every 50 iterations. It uses the Adam optimizer with an initial learning rate of $10^{-1}$. Scene parameters are dataset-dependent [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Reduced compared to single-resolution training due to early convergence on global structure.
- **Memory Complexity:** Lower during the coarse stage due to the smaller bounding box and scalar features.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales well with dynamic scenes by managing computational load through progressive data addition.
- **Strengths:** Stabilizes training, filters empty space, and accelerates convergence.
- **Limitations:** Requires careful tuning of the canonical time step and bounding box estimation.
- **When to Use:** Dynamic scenes with significant temporal variation.
- **When Not to Use:** Static scenes where single-resolution training is sufficient.

## Background
Originated in the context of dynamic radiance field optimization, specifically for novel view synthesis of dynamic scenes. The method was proposed to address the challenges of training radiance fields on temporally varying data by leveraging a coarse-to-fine approach [S1]. It builds on earlier work in radiance field optimization and progressive training strategies.

## Sources
[S1] Forward Flow for Novel View Synthesis of Dynamic Scenes, https://doi.org/10.1109/iccv51070.2023.01468, fetched 2026-10-26
[S2] ActiveRMAP: Radiance Field for Active Mapping And Planning, https://doi.org/10.48550/arxiv.2211.12656, fetched 2026-10-26 (unofficial)
