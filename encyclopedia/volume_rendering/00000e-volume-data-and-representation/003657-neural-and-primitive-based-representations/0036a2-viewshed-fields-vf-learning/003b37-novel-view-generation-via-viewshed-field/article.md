# Novel View Generation via Viewshed Field

Also called: Inverse Normalizing Flow Sampling, VF Sampling

## Overview
This method generates novel views by treating the generation of high-confidence oriented points as a generative process using Normalizing Flows (NF) [S1]. It samples from a learned latent Gaussian distribution, inverts the mapping to obtain oriented points $(x, d)$ with high Viewshed Field (VF) scores, and reconstructs camera origins to render new perspectives [S1]. It differs from standard NeRF rendering by explicitly sampling the latent space of valid viewing configurations rather than traversing a ray, ensuring generated views are geometrically consistent with the learned VF representation [S1].

## How it works
**Inputs:**
*   A trained Normalizing Flows model $f$ that maps oriented points to a latent space, and its inverse $f^{-1}$ [S1].
*   A trained Viewshed Field (VF) representation [S1].
*   Camera parameters for the target novel view direction $d$ (optional, if direction is sampled; otherwise fixed) [S1].

**Outputs:**
*   A rendered image of the novel view [S1].

**Data Structures:**
*   Latent space $Z$: 6-dimensional Gaussian distribution $\mathcal{N}(0, I)$ [S1].
*   Oriented point: Tuple $(x, d)$ where $x \in \mathbb{R}^3$ is the 3D position and $d \in \mathbb{R}^3$ is the unit view direction vector [S1].
*   Camera origin: Vector $o \in \mathbb{R}^3$ [S1].

**Parameters:**
*   $K$: Number of samples drawn from the latent space $Z$. Not given in the sources.
*   $N$: Number of top-ranked oriented points selected based on log probability. Not given in the sources.
*   Threshold: Value applied to the VF to create a 2D mask. Not given in the sources.
*   Architecture: Real-NVP is used for the Normalizing Flows due to the low input dimensionality (6D) [S1].

**Steps:**
1.  **Sample Latent Points:** Draw $K$ independent samples $z_i$ from the 6-dimensional standard Gaussian distribution $Z \sim \mathcal{N}(0, I)$ [S1].
2.  **Inverse Mapping:** For each sample $z_i$, compute the corresponding oriented point $(x_i, d_i)$ using the inverse normalizing flow function:
    $$
    (x_i, d_i) = f^{-1}(z_i)
    $$
    [S1]
3.  **Score and Select:** Calculate the log probability $\log(p_x(x_i))$ for each oriented point using the VF model [S1]. Select the top-$N$ oriented points with the highest log probabilities [S1].
4.  **Reconstruct Camera Origin:** For each selected oriented point $(x, d)$, reconstruct the camera origin $o$ using the following equation:
    $$
    o = x - \text{depth} \cdot d
    $$
    [S1]
    *Note: The specific formula for "depth" is not provided in the sources, only referenced as part of Equation 5 [S1]. [G] In the context of VF, depth typically corresponds to the distance from the point $x$ to the surface along the direction $d$, or a fixed offset if $x$ is already on the surface. If $x$ is a point on the surface and $d$ is the viewing direction, $o$ is the camera position looking at $x$. If $x$ is arbitrary, depth must be derived from the VF geometry.*
5.  **Render View:** Render the scene from the camera positioned at $o$ facing direction $d$ [S1].
6.  **Masking (Optional/Refinement):** Render the Viewshed Field for the generated view. Apply a threshold to the VF values to create a 2D viewshed mask. Sample pixels according to this mask to ensure only high-confidence regions are included in the final output [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_novel_view(vf_model, nf_inverse, K, N, threshold):
    """
    Generates a novel view by sampling high-score oriented points from the VF latent space.
    
    Args:
        vf_model: The trained Viewshed Field model (provides log probability).
        nf_inverse: The inverse Normalizing Flow function f^-1.
        K: Number of latent samples to draw.
        N: Number of top points to select.
        threshold: Threshold for the 2D viewshed mask.
        
    Returns:
        rendered_image: The final rendered novel view.
    """
    # Step 1: Sample from 6D Gaussian latent space
    # Z ~ N(0, I)
    z_samples = sample_gaussian_6d(K)
    
    # Step 2: Inverse mapping to oriented points (x, d)
    oriented_points = []
    for z in z_samples:
        x, d = nf_inverse(z)
        oriented_points.append((x, d))
        
    # Step 3: Select top-N points by log probability
    # Calculate log probability for each oriented point
    scores = []
    for x, d in oriented_points:
        # log_px(x) is derived from the VF model's likelihood estimation
        score = vf_model.log_probability(x, d)
        scores.append(score)
        
    # Sort by score descending and take top N
    sorted_indices = argsort_descending(scores)
    top_n_indices = sorted_indices[:N]
    selected_points = [oriented_points[i] for i in top_n_indices]
    
    # Step 4 & 5: Reconstruct camera and render
    # Note: 'depth' is not explicitly defined in the source text beyond Eq 5.
    # [G] Assuming depth is the distance from the oriented point to the camera
    # along the viewing direction, or a fixed value if x is on the surface.
    # If x is a point on the surface and d is the view direction, 
    # the camera origin o is located at x - depth * d.
    
    final_image = None
    
    for x, d in selected_points:
        # Reconstruct camera origin
        # o = x - depth * d
        # Since depth is not given, we assume a standard reconstruction where
        # the camera is placed at a distance from the point x in direction -d.
        # [G] Common practice: depth might be inferred from VF or set to a small epsilon
        # if x is already on the surface. Here we leave depth as a variable.
        depth = get_depth_from_vf(x, d) # Placeholder for missing depth logic
        o = x - depth * d
        
        # Render view from origin o facing direction d
        # This involves ray marching through the VF/NeRF
        image = render_view(vf_model, o, d)
        
        # Step 6: Apply viewshed mask
        # Render VF for this view
        vf_map = render_vf_field(vf_model, o, d)
        mask = vf_map > threshold
        
        # Sample pixels according to mask
        masked_image = apply_mask(image, mask)
        
        # Combine or select best view
        # The source implies generating a view from the point, potentially aggregating
        # or selecting one. We return the last processed or aggregate.
        final_image = masked_image
        
    return final_image

def sample_gaussian_6d(K):
    """Samples K points from a 6-dimensional standard normal distribution."""
    return np.random.randn(K, 6)

def get_depth_from_vf(x, d):
    """
    [G] Helper to estimate depth. 
    In VF-NeRF, depth is often associated with the distance to the surface.
    If x is an oriented point on the surface, depth might be 0 or a small offset.
    If x is in free space, depth is the distance to the nearest surface along d.
    This function is a placeholder as the exact depth calculation is not in the source.
    """
    # Simplified assumption: if x is on surface, depth is 0.
    # If the VF provides a distance field, use it.
    return 0.0 
```

## Subtopics

### Novel View Generation via Normalizing Flows
A specific application of the method where the Normalizing Flows architecture (specifically Real-NVP) is used to map high-value VF points to a Gaussian distribution [S1]. It is chosen for its efficiency with low-dimensional inputs (6D) [S1]. The process involves sampling from the 6D Gaussian, recovering the oriented point $(\tilde{x}, \tilde{d})$ via inverse mapping, and reconstructing the camera origin using Equation 5 ($o = \tilde{x} - \text{depth} \cdot \tilde{d}$) [S1]. It is used to initialize registration or generate novel views where standard ray marching might fail to find high-confidence configurations [S1].

## Operating characteristics
*   **Time Complexity:** Proportional to the number of samples $K$ and the complexity of the Normalizing Flow inverse mapping and VF evaluation [S1]. The computational effort is proportional to the dimension of the data (6D) [S1].
*   **Memory Complexity:** Depends on the storage of the VF and the NF model parameters [S1].
*   **Strengths:** Enables generation of novel views from high-probability oriented points without iterative root-finding or extensive sampling required by standard implicit shapes [S1]. Useful for initializing registration processes by generating 3D point clouds [S1].
*   **Limitations:** Requires a trained Normalizing Flows model alongside the VF [S1]. The specific depth reconstruction formula (Equation 5) relies on an undefined "depth" parameter in the provided text [S1]. High viewshed score is not guaranteed for all pixels in the rendered view, necessitating masking [S1].
*   **When to use:** When generating novel views that are geometrically consistent with the VF representation, or for initializing rigid registration between scenes using VF-generated point clouds [S1].
*   **When not to use:** When high-quality novel views are required for all pixels without post-processing masking, or when the depth information is unavailable and cannot be inferred [S1].

## Background
The method originates from "VF-NeRF: Viewshed Fields for Rigid NeRF Registration" [S1]. It addresses the challenge that VF is an implicit function and does not directly provide oriented points for virtual camera placement [S1]. By treating the problem as a generative process using Normalizing Flows (NF) [12], the method maps high-value VF points to a Gaussian distribution in latent space during NeRF training [S1]. This allows for the sampling of high-value VF points to direct virtual cameras, aiding in novel view synthesis and registration initialization [S1]. The approach builds on prior work in NeRF registration such as DRegNeRF [6] and BARF [26], but uses VF and NF for explicit geometric guidance [S1].

## Sources
[S1] VF-NeRF: Viewshed Fields for Rigid NeRF Registration, https://doi.org/10.1007/978-3-031-73027-6_10, fetched 2026-10-26
