# Online Incremental NeRF Optimization
Also called: Incremental NeRF Training, Online NeRF

## Overview
Online Incremental NeRF Optimization is a training paradigm where the radiance field representation is updated continuously as new camera views are captured, rather than optimizing a static dataset offline. It solves the problem of active 3D reconstruction in dynamic or large-scale environments where data arrives sequentially. Unlike offline methods such as DVGO, which optimize based on a fixed, pre-collected set of images, this method plans the next best views and perceives new observations based on the current scene representation, adding them to the training set incrementally [S1, unofficial].

## How it works
The method operates as an iterative loop that integrates new viewpoints while managing the risk of catastrophic forgetting.

**Inputs and Outputs:**
- **Inputs:** Current radiance field representation (feature grids and decoder), current camera poses, new captured image and its pose.
- **Outputs:** Updated radiance field representation.

**Data Structures:**
- **Feature Grids:** Spatially localized feature storage (e.g., hashgrid or dense grid) that mitigates forgetting by isolating regions [S2, unofficial].
- **Decoder:** A shared MLP (Multi-Layer Perceptron) that maps features to color and density. In standard implementations, this is updated globally, which is a primary source of forgetting [S2, unofficial].

**Steps:**
1. **Perceive:** Capture a new observation (image and pose) based on the current scene representation. The agent plans the next best view to maximize information gain (e.g., using entropy-based metrics) [S1, unofficial].
2. **Integrate:** Add the new observation to the training set.
3. **Optimize:** Update the radiance field parameters using the new observation.
   - If using a standard NeRF, the shared MLP weights are updated globally via gradient descent.
   - If using the "Frozen Decoder" variant (see Subtopics), the decoder weights remain frozen, and only the feature grids are optimized [S2, unofficial].
4. **Repeat:** Continue to step 1 for subsequent views.

**Formulas:**
The optimization minimizes the photometric loss between the rendered image $\hat{I}$ and the ground truth image $I$:
$$ \mathcal{L} = \| I - \hat{I} \|^2 $$
Where $\hat{I}$ is computed via volume rendering from the current radiance field parameters [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Catastrophic Forgetting:** In vanilla NeRF, every optimization step updates the shared MLP weights globally. This causes previously reconstructed regions, which are no longer supervised by new views, to degrade [S2, unofficial].
- **Memory Usage:** Exploring large environments results in a high number of captured images, requiring strategies to discard older frames or manage memory [S2, unofficial].
- **Hashgrid Dictionary:** Even with grid-based representations, forgetting can occur due to the hashgrid dictionary itself, not just the decoder [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialize radiance field with feature grids and decoder
radiance_field = initialize_radiance_field()
training_data = []

while not scene_complete:
    # 1. Plan next best view based on current representation
    # Uses entropy or uncertainty metrics to select viewpoint
    next_pose = plan_next_best_view(radiance_field)
    
    # 2. Perceive new observation
    new_image, new_pose = capture_observation(next_pose)
    
    # 3. Add to training set
    training_data.append((new_image, new_pose))
    
    # 4. Optimize representation
    # Sample rays from new image
    rays = generate_rays(new_image, new_pose)
    
    # Render current prediction
    predicted_color = volume_render(radiance_field, rays)
    
    # Compute loss
    loss = compute_photometric_loss(predicted_color, new_image)
    
    # Update parameters
    # Note: In standard online NeRF, this updates both grids and MLP
    # In Frozen Decoder variant, MLP gradients are blocked
    radiance_field.update(loss)
```

## Subtopics

### Incremental Reconstruction with Frozen Decoder
A strategy for online incremental reconstruction to address catastrophic forgetting, where the model integrates new viewpoints while preserving previously learned information. It employs pre-trained, scene-agnostic decoders that remain frozen during the feature grids optimization. This design enables new grid cells to represent unseen areas while preserving the integrity of already reconstructed regions. It is chosen when catastrophic forgetting in the shared MLP is a critical issue, though it may limit the adaptability of the color/density mapping to new lighting conditions [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of new views added, but each step requires full gradient computation through the network.
- **Memory Complexity:** Depends on the size of the feature grids and the number of stored views. Large environments require significant memory or frame discarding strategies [S2, unofficial].
- **Strengths:** Allows for real-time or near-real-time reconstruction as the agent explores. Can build better 3D models with a limited number of viewpoints by actively selecting informative views [S1, unofficial].
- **Limitations:** Susceptible to catastrophic forgetting in vanilla implementations. Performance is bounded by the "ground-truth" of offline methods like DVGO if used as a reference [S1, unofficial].
- **When to use:** Active mapping, robotic navigation, and large-scale scene exploration where data arrives sequentially.
- **When not to use:** Offline processing of static, pre-collected datasets where global optimization is feasible.

## Background
The concept emerged from the need to adapt NeRFs to real-world scenarios involving imperfect data, large-scale scenes, and incremental reconstruction. It contrasts with offline methods like DVGO, which optimize based on a fixed set of images. The approach is closely related to active vision and active mapping, where the camera pose is planned to maximize information gain [S1, unofficial]. The issue of catastrophic forgetting in neural networks when updating with new data is a well-known challenge in continual learning, which this method addresses through spatially localized feature storage and potentially frozen decoders [S2, unofficial].

## Sources
[S1] ActiveRMAP: Radiance Field for Active Mapping And Planning, https://doi.org/10.48550/arxiv.2211.12656, fetched 2026-10-26 (unofficial)
[S2] From reconstruction to reasoning : neural radiance fields for comprehensive 3D scene understanding, https://theses.hal.science/tel-05693528v1, fetched 2026-10-26 (unofficial)
