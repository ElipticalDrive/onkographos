# Uncertainty Guided Policy with Region Clustering

Also called: Active View Selection via Entropy, Coarse-to-Fine Reconstruction Policy

## Overview
This method is an active view selection policy for robotic 3D reconstruction using Neural Radiance Fields (NeRF). It selects the next best views by maximizing information gain, approximated via the mean entropy of a candidate view's uncertainty map. It distinguishes itself from global greedy selection by enforcing spatial diversity through region clustering, preventing the selection of redundant, similar viewpoints that cause overfitting, while avoiding the slow coverage of single-view-per-iteration strategies.

## How it works
**Inputs:**
- A set of candidate camera poses $\mathcal{C}$ distributed on a hemisphere surrounding the object.
- A current NeRF model $\mathcal{N}$ trained on an initial dataset $\mathcal{D}_{init}$.
- A target image resolution for uncertainty evaluation.

**Outputs:**
- A set of selected camera poses $\mathcal{S}$ to be captured in the current iteration.
- An updated NeRF model $\mathcal{N}'$.

**Data Structures:**
- **View Space:** A hemisphere divided into 13 regions: 12 sections (clusters) and 1 middle circle (excluded from selection).
- **Sections:** The hemisphere is split into an upper half and a lower half relative to the middle circle. Each half is divided into 6 groups based on azimuthal angle $\phi$, resulting in 12 sections $R_1, \dots, R_{12}$.
- **Candidate Poses:** 30 poses per horizontal circle, 5 circles total, yielding 150 candidates.

**Parameters:**
- **Candidate Poses:** 150 total (30 per circle $\times$ 5 circles).
- **Initialization Images:** 6 for synthetic data, 15 for real-world data, sampled from the middle circle.
- **Sampling:** 64 samples for coarse network, 128 for fine network [S1].
- **Downsampling:** Images are downsampled by a factor of 4 for entropy evaluation [S1].

**Steps:**
1. **Initialization:** Train an initial NeRF model $\mathcal{N}_0$ using images from the middle circle (6 images for synthetic, 15 for real-world) [S1].
2. **Region Definition:** Divide the hemisphere into 12 sections for clustering. The middle circle is excluded from selection regions [S1].
3. **Iteration Loop:**
   a. For each of the 12 sections $R_i$:
      i. Identify all candidate poses $c \in \mathcal{C}$ that fall within section $R_i$.
      ii. For each candidate pose $c$, render the scene using $\mathcal{N}$ to obtain the weight distribution $w$ for each pixel.
      iii. Compute the entropy $H(c, p)$ for each pixel $p$ in the rendered image. The entropy is derived from the ray-based volumetric uncertainty estimation [S1].
      iv. Calculate the mean entropy $\bar{H}(c)$ for the entire image by averaging $H(c, p)$ across all pixels [S1].
      v. Select the candidate pose $c^*_i$ with the highest mean entropy $\bar{H}(c)$ within section $R_i$.
   b. Collect the 12 selected views $\mathcal{S} = \{c^*_1, \dots, c^*_{12}\}$ [S1].
   c. Update the training dataset $\mathcal{D} \leftarrow \mathcal{D} \cup \mathcal{S}$.
   d. Refine the NeRF model $\mathcal{N}$ using the updated dataset $\mathcal{D}$, initializing from the previous model state [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def uncertainty_guided_policy_with_region_clustering(candidate_poses, initial_images, num_sections=12):
    # Step 1: Initialization
    # Train initial NeRF model using images from the middle circle
    nerf_model = train_nerf(initial_images)
    
    # Step 2: Define regions
    # Divide hemisphere into 12 sections (upper/lower halves, 6 azimuthal groups each)
    # Exclude middle circle from selection regions
    sections = define_hemisphere_sections(candidate_poses, num_sections)
    
    selected_views = []
    
    # Step 3: Iterative selection
    while not convergence_criteria_met(nerf_model):
        current_selection = []
        
        for section in sections:
            # Get candidates in this section
            candidates_in_section = [c for c in candidate_poses if c in section]
            
            best_candidate = None
            max_mean_entropy = -1
            
            for candidate in candidates_in_section:
                # Render scene with current NeRF model
                # Downsample images by factor of 4 for speed [S1]
                rendered_image = render(nerf_model, candidate)
                
                # Compute entropy for each pixel
                # H is derived from ray-based volumetric uncertainty [S1]
                entropies = compute_pixel_entropy(rendered_image)
                
                # Mean entropy of the image
                mean_entropy = sum(entropies) / len(entropies)
                
                if mean_entropy > max_mean_entropy:
                    max_mean_entropy = mean_entropy
                    best_candidate = candidate
            
            current_selection.append(best_candidate)
        
        # Add selected views to dataset
        new_images = capture_images(current_selection)
        dataset.update(new_images)
        
        # Refine NeRF model
        nerf_model = refine_nerf(nerf_model, dataset)
        
        selected_views.extend(current_selection)
        
        # Check convergence (e.g., F-score threshold or max iterations)
        
    return selected_views, nerf_model

def compute_pixel_entropy(rendered_image):
    # Returns entropy values for each pixel based on weight distribution
    # Implementation details depend on the specific ray-based volumetric uncertainty estimator [S1]
    pass
```

**Edge Cases and Pitfalls:**
- **Overfitting:** Selecting top-k most uncertain viewpoints without region constraints or model updates may result in choosing similar viewpoints, leading to overfitting [S1].
- **Inefficiency:** Selecting only one view globally per iteration without spatial constraints results in much longer acquisition processes to cover object geometry [S1].
- **Precision:** Entropy calculation requires accurate weight distributions from the NeRF. Downsampling by a factor of 4 is used to speed up the process, which may affect precision [S1].

## Operating characteristics
- **Time Complexity:** Depends on the number of candidate poses (150) and the number of sections (12). For each iteration, 12 candidates are evaluated. Each evaluation requires rendering the scene and computing entropy for all pixels. Downsampling by a factor of 4 reduces computational cost [S1].
- **Memory Complexity:** Requires storing the NeRF model and the training dataset. The dataset grows with each iteration.
- **Performance:** The method is evaluated on synthetic and real-world data. It achieves high-quality 3D meshes with fewer images than dense sampling. The F-score metric is used for evaluation [S1].
- **Strengths:** Ensures diverse view selection, prevents overfitting, and accelerates coverage compared to single-view strategies.
- **Limitations:** Requires a hemisphere view space definition. Performance depends on the quality of the initial coarse reconstruction.

## Background
The method was proposed in "Uncertainty Guided Policy for Active Robotic 3D Reconstruction Using Neural Radiance Fields" [S1]. It builds on the concept of active robotic 3D reconstruction using NeRF, leveraging ray-based volumetric uncertainty estimation to guide view selection. The approach addresses the challenge of efficiently selecting novel views to improve 3D reconstruction quality.

## Sources
[S1] Uncertainty Guided Policy for Active Robotic 3D Reconstruction Using Neural Radiance Fields, https://doi.org/10.1109/lra.2022.3212668, fetched 2026-10-26
