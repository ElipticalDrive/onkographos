# Online Incremental NeRF Optimization
Also called: Incremental NeRF Training, Online NeRF

## Overview
Online Incremental NeRF Optimization addresses the problem of reconstructing 3D scenes from a continuous stream of incoming camera views, rather than a fixed, pre-collected dataset. It differs from offline methods like DVGO [S1, unofficial] by updating the scene representation dynamically as new observations are acquired, enabling active exploration where the agent plans the next best view based on the current model's uncertainty.

## How it works
**Inputs and Outputs:**
*   **Inputs:** A current scene representation (e.g., feature grids, MLP), a set of previously observed images with poses, and a newly captured image with its pose.
*   **Outputs:** An updated scene representation and potentially a plan for the next best view.

**Data Structures:**
The representation typically consists of spatially localized feature grids (e.g., hash grids) and a decoder (MLP). The specific layout depends on the underlying NeRF variant (e.g., Instant-NGP, DVGO), but generally involves a mapping from 3D coordinates to feature vectors.

**Steps:**
1.  **Perception:** The agent captures a new observation (image and pose) based on the current scene representation. This may involve an active planning step to select the "next best view" that maximizes information gain or minimizes uncertainty [S1, unofficial].
2.  **Incremental Update:** The new observation is added to the training set. The scene representation is optimized using this new data.
3.  **Re-optimization:** The model parameters are updated to minimize the rendering loss on the new view while attempting to preserve previously learned information. This often involves adjusting learning rates or using specific architectural strategies to mitigate catastrophic forgetting [S2, unofficial].

**Formulas:**
Let $R$ be the radiance field, $I_{new}$ be the new image, and $P_{new}$ be its pose. The optimization minimizes:
$$ \mathcal{L} = \mathbb{E}_{(x, \omega) \sim \Omega} [ \| R(x, \omega) - I_{new}(\omega) \|^2 ] $$
where $\Omega$ is the set of rays cast through the new image. The representation $R$ is updated incrementally.

**Parameters:**
*   **Learning Rate:** Typically adjusted during re-optimization to accommodate new capacity without destroying previous knowledge. Not given in the sources.
*   **Entropy Threshold:** Used in active planning to determine when to stop or select views. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Catastrophic Forgetting:** In vanilla NeRF, global MLP updates cause previously reconstructed regions (not supervised by new views) to degrade [S2, unofficial].
*   **Memory Usage:** Large environments result in high memory usage, potentially requiring discarding of older frames [S2, unofficial].
*   **Hashgrid Dictionary:** Forgetting can still occur due to hash collisions or dictionary updates in grid-based representations [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def online_incremental_nerf_update(scene_rep, new_image, new_pose):
    """
    Updates the scene representation with a new observation.
    scene_rep: Current NeRF representation (grids + decoder)
    new_image: Tensor of shape [H, W, 3]
    new_pose: Camera pose matrix [4, 4]
    """
    # 1. Cast rays through the new image
    rays = cast_rays(new_image.shape, new_pose)
    
    # 2. Render current prediction
    predicted_colors = render_rays(scene_rep, rays)
    
    # 3. Compute loss
    loss = mse_loss(predicted_colors, new_image)
    
    # 4. Update representation
    # Note: Specific optimization strategy (e.g., frozen decoder) 
    # depends on the variant. Here we assume a standard gradient step.
    gradients = compute_gradients(loss, scene_rep.parameters())
    update_parameters(scene_rep.parameters(), gradients)
    
    return scene_rep
```

## Subtopics

### Incremental Reconstruction with Frozen Decoder
A strategy to address catastrophic forgetting in online incremental reconstruction. It employs pre-trained, scene-agnostic decoders that remain frozen during feature grid optimization. This enables new grid cells to represent unseen areas while preserving the integrity of already reconstructed regions. It is chosen when memory constraints or the risk of forgetting previously learned geometry/appearance is high.

## Operating characteristics
*   **Time Complexity:** Depends on the underlying NeRF variant (e.g., $O(1)$ per ray for hash grids). Re-optimization cost scales with the number of parameters and the learning rate schedule.
*   **Memory Complexity:** Scales with the size of the feature grids and the number of stored frames if not discarded. Large environments require significant memory [S2, unofficial].
*   **Strengths:** Enables real-time or near-real-time reconstruction from streaming data; supports active exploration.
*   **Limitations:** Susceptible to catastrophic forgetting without specific architectural interventions (e.g., frozen decoders); performance may degrade if older frames are discarded without careful management.
*   **When to use:** Active mapping, SLAM, or scenarios where data arrives sequentially and offline retraining is not feasible.
*   **When not to use:** When a fixed, high-quality offline reconstruction is required and all data is available upfront.

## Background
Originating from the need to adapt NeRFs to real-world, dynamic scenarios, this method builds on the principles of active vision and incremental learning. It differs from offline methods like DVGO by optimizing the representation online as new views are collected [S1, unofficial]. The challenge of catastrophic forgetting in neural networks when updating with new data is a key driver for variants like the frozen decoder approach [S2, unofficial].

## Sources
[S1] ActiveRMAP: Radiance Field for Active Mapping And Planning, https://doi.org/10.48550/arxiv.2211.12656, fetched 2026-10-26 (unofficial)
[S2] From reconstruction to reasoning : neural radiance fields for comprehensive 3D scene understanding, https://theses.hal.science/tel-05693528v1, fetched 2026-10-26 (unofficial)
