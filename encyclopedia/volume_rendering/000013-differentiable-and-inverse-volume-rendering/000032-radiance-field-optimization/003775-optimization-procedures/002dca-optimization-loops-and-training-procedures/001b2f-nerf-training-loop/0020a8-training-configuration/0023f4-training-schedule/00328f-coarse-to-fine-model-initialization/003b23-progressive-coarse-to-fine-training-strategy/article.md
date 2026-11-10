# Progressive Coarse-to-Fine Training Strategy
Also called: Training Strategy
## Overview
A training protocol for dynamic radiance fields that combines temporal progression with spatial resolution scaling to stabilize optimization and improve geometry. It sets itself apart from single-resolution or static-temporal approaches by decoupling global structural coherence from high-frequency detail reconstruction, ensuring the model converges to a valid global geometry before committing computational resources to fine-grained features [S1].
## How it works
The method consists of two distinct phases: a Coarse Stage and a Fine Stage, applied sequentially.

**Inputs:**
- A set of dynamic scene images spanning a timeline.
- A canonical frame index (reference time).
- A proxy geometry estimator (for bounding box computation).

**Outputs:**
- A final dynamic radiance field model (from the Fine Stage).

**Data Structures:**
- **Color Feature:** In the Coarse Stage, this is the color itself (scalar/vector per sample). In the Fine Stage, this is a high-dimensional feature vector.
- **Bounding Box:** Computed in the Coarse Stage using proxy geometry to filter empty space.

**Numbered Steps:**

1. **Progressive Temporal Scheduling:**
   - Initialize the training set with images close to the canonical time.
   - Iteratively add images with progressively farther time steps.
   - Continue until all available images are included in the training set.

2. **Coarse Stage Training:**
   - **Feature Definition:** Set the color feature to be the color itself, ignoring ray direction dependency.
   - **Geometry Filtering:** Compute a smaller bounding box using proxy geometry learned during this stage. This filters a large portion of empty space.
   - **Optimization:** Train the model using the temporally progressive schedule within this restricted bounding box.

3. **Fine Stage Training:**
   - **Initialization:** Initialize the model using the weights from the Coarse Stage.
   - **Feature Definition:** Set the color features to be high-dimensional features.
   - **Directional Modeling:** Model ray direction dependency using a specific network component, denoted as $F_{\theta4}$.
   - **Optimization:** Train the final model using the full set of images and the refined bounding box from the Coarse Stage.

**Formulas:**
The loss function used during training (specifically cited in the context of the overall method in [S1]) is:
$$
\mathcal{L} = w_1 \mathcal{L}_{ptc} + w_2 \mathcal{L}_{bg} + w_3 \mathcal{L}_{flow} + w_4 \mathcal{L}_{vdiff} + w_5 \mathcal{L}_{tv}(\mathbf{V}_{\text{grid}}^{\sigma_{\text{Can}}}) + w_6 \mathcal{L}_{tv}(\mathbf{V}_{\text{grid}}^{\text{flow}_t}) + w_7 \mathcal{L}_{tv}(\mathbf{D})
$$
Where:
- $\mathcal{L}_{ptc}$: Point-wise color loss.
- $\mathcal{L}_{bg}$: Background loss.
- $\mathcal{L}_{flow}$: Flow consistency loss.
- $\mathcal{L}_{vdiff}$: Velocity difference loss.
- $\mathcal{L}_{tv}$: Total variation loss.
- $\mathbf{V}_{\text{grid}}^{\sigma_{\text{Can}}}$: Canonical space density grid.
- $\mathbf{V}_{\text{grid}}^{\text{flow}_t}$: Flow grid at time $t$.
- $\mathbf{D}$: Displacement field.
- $w_1 \dots w_7$: Weighting coefficients.

**Parameters:**
- $F_{\theta4}$: The network component modeling ray direction dependency in the Fine Stage.
- Canonical time: The reference frame for the timeline.

**Edge Cases and Pitfalls:**
- **Empty Space:** Without the proxy geometry bounding box in the Coarse Stage, training may waste resources on empty regions.
- **Directional Dependency:** Ignoring ray directions in the Coarse Stage simplifies learning but may lead to view-dependent artifacts if not corrected in the Fine Stage.
- **Temporal Order:** Adding far-time images too early may destabilize optimization due to large motion/disocclusions.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def progressive_coarse_fine_training(images, canonical_time_idx):
    # Step 1: Progressive Temporal Scheduling
    # Sort images by distance from canonical_time_idx
    sorted_images = sort_by_temporal_distance(images, canonical_time_idx)
    
    # Initialize model with Coarse Stage configuration
    model = initialize_model(color_feature_type='color', direction_dependency=False)
    
    # Coarse Stage
    for batch in progressive_batches(sorted_images):
        # Forward pass with color-only features
        rendered_colors = model.forward(batch.rays, feature_type='color')
        # Compute loss (includes flow, tv, etc. as per [S1])
        loss = compute_loss(rendered_colors, batch.targets, model)
        # Backpropagate
        model.update(loss)
        
    # Compute proxy geometry and bounding box
    proxy_geometry = model.extract_proxy_geometry()
    bounding_box = compute_bounding_box(proxy_geometry)
    
    # Fine Stage
    # Re-initialize or update model for Fine Stage
    fine_model = initialize_fine_model(
        weights=model.weights,
        color_feature_type='high_dim',
        direction_dependency=True,
        bounding_box=bounding_box
    )
    
    # Train with all images
    for batch in all_images_batches(images):
        # Forward pass with high-dim features and direction dependency via F_theta4
        rendered_colors = fine_model.forward(batch.rays, feature_type='high_dim', use_direction=True)
        loss = compute_loss(rendered_colors, batch.targets, fine_model)
        fine_model.update(loss)
        
    return fine_model
```
## Subtopics

### [Two-Stage Training Pipeline](11433-two-stage-training-pipeline.md)
A training strategy that first recovers global shape at low resolution, then refines fine details at high resolution. It sets itself apart from single-resolution approaches by decoupling structural coherence from high-frequency detail reconstruction, ensuring the model converges to a valid global geometry before committing computational resources to fine-grained features.

## Operating characteristics
- **Time Complexity:** Higher than single-stage training due to two sequential phases, but reduces overall convergence time by stabilizing early optimization.
- **Memory Complexity:** Reduced in the Coarse Stage due to the smaller bounding box filtering empty space.
- **Strengths:** Improves geometry accuracy by establishing global structure first; handles dynamic scenes with large motion better by progressive temporal addition.
- **Limitations:** Requires careful tuning of the temporal progression schedule; the proxy geometry estimation must be robust.
- **When to use:** Dynamic scene reconstruction where global shape is critical and initial optimization is unstable.
- **When not to use:** Static scenes or scenes with minimal motion where single-resolution training suffices.
## Background
Originated in the context of dynamic radiance field optimization, specifically for novel view synthesis of dynamic scenes. The method was proposed to address the instability of training neural radiance fields on dynamic data by decoupling temporal and spatial complexity.
## Sources
[S1] Forward Flow for Novel View Synthesis of Dynamic Scenes, https://doi.org/10.1109/iccv51070.2023.01468, fetched 2026-10-26
