# Canonical Neural Radiance Field for X-ray Tomography

Also called: Canonical volume fitting, NeRF for X-ray

## Overview
Canonical Neural Radiance Field for X-ray Tomography is a method that reconstructs the 3D density field of a static object (canonical volume) using a Neural Radiance Field (NeRF) trained on rich projection data, serving as a spatial prior for dynamic reconstruction [S1, unofficial]. It differs from full spatio-temporal networks by decoupling the static canonical volume $(x_0, y_0, z_0) \rightarrow (c, \sigma)$ from the deformation field, addressing the poor performance observed when simply adding time as a direct input to conventional NeRFs [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** X-ray projection data. Specifically, "rich" many-projection data acquired at static time points (e.g., $t=0$ and $t=1$) and "sparse" few-projection data acquired during dynamic events at intermediate times [S1, unofficial].
*   **Outputs:** A canonical volume representation defined by the neural network, which maps canonical coordinates to density and color, and a deformation field mapping dynamic coordinates to canonical coordinates [S1, unofficial].

**Data Structures:**
The method utilizes a Neural Radiance Field architecture. While specific network layer dimensions are not provided in the sources, the core data structure is a function approximator $\mathcal{F}_\theta$ parameterized by weights $\theta$ [S1, unofficial].
*   **Canonical Volume:** A mapping from canonical spatial coordinates $(x_0, y_0, z_0)$ to volume density $\sigma$ and color $c$.
*   **Deformation Field:** A mapping from dynamic spatial coordinates $(x, y, z)$ and time $t$ to canonical coordinates $(x_0, y_0, z_0)$ [S1, unofficial].

**Physical Model:**
The rendering process is governed by the Beer-Lambert law, where X-ray attenuation follows:
$$ I = I_0 \exp \left( - \int \mu(x) dx \right) $$
where $I$ is the transmitted intensity, $I_0$ is the initial intensity, and $\mu(x)$ is the attenuation coefficient [S1, unofficial].

The projection $p_\theta(r)$ at angle $\theta$ is modeled as the Radon transform of the density function $\mu(x,y)$:
$$ p_\theta(r) = \int_{x,y} \mu(x,y) \delta(x \cos \theta + y \sin \theta - r) dx dy $$
where $\delta$ is the Dirac delta function, and $(x,y)$ are coordinates in the slice perpendicular to the beam [S1, unofficial].

**Parameters:**
*   **Number of Projections:** For conventional CT reconstruction of the canonical volume, a typical value is 3000 projections. This ensures the maximum displacement of sample points between projections is within one voxel [S1, unofficial].

**Algorithm Steps:**
1.  **Pre-training (Canonical Volume Fitting):**
    *   Acquire rich, many-projection X-ray data at static time points (e.g., $t=0$ and $t=1$) [S1, unofficial].
    *   Train the NeRF to reconstruct the canonical volume from this rich data. This step establishes the baseline static geometry and appearance [S1, unofficial].
2.  **Deformation Field Calculation:**
    *   Acquire sparse, few-projection X-ray data at intermediate dynamic time points [S1, unofficial].
    *   Optimize the deformation field parameters to map the dynamic coordinates of the sparse data to the pre-trained canonical volume, minimizing the difference between rendered projections and observed sparse projections [S1, unofficial].
3.  **Spatio-Temporal Reconstruction:**
    *   Combine the fixed canonical volume and the learned deformation field to reconstruct the full field spatio-temporal information for any time $t$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for Canonical Neural Radiance Field for X-ray Tomography
# Based on [S1, unofficial]

def train_canonical_volume(rich_projections_t0, rich_projections_t1):
    """
    Step 1: Pre-training. Reconstruct the canonical volume using rich data.
    rich_projections: List of (angle, projection_image) tuples.
    """
    # Initialize NeRF parameters theta
    theta = initialize_neural_network()
    
    # Loss function based on Beer-Lambert law and Radon transform consistency
    def loss_fn(theta, projections):
        total_loss = 0
        for angle, observed_projection in projections:
            # Render projection from current NeRF state
            rendered_projection = render_neural_projection(theta, angle)
            # Compare rendered vs observed (e.g., MSE)
            total_loss += mse(rendered_projection, observed_projection)
        return total_loss

    # Optimize theta to fit the canonical volume
    theta = optimize(theta, loss_fn, rich_projections_t0 + rich_projections_t1)
    return theta

def train_deformation_field(canonical_theta, sparse_projections_dynamic):
    """
    Step 2: Calculate deformation field from sparse data.
    canonical_theta: Pre-trained weights for the static volume.
    sparse_projections_dynamic: List of (angle, projection_image, time_t) tuples.
    """
    # Initialize deformation field parameters phi
    phi = initialize_deformation_field()
    
    # Loss function comparing rendered dynamic projections to observed sparse data
    def loss_fn(phi, canonical_theta, dynamic_data):
        total_loss = 0
        for angle, observed_projection, time_t in dynamic_data:
            # Render projection using canonical volume and deformation field
            # The deformation field maps dynamic coordinates (x,y,z,t) to canonical (x0,y0,z0)
            rendered_projection = render_dynamic_projection(canonical_theta, phi, angle, time_t)
            total_loss += mse(rendered_projection, observed_projection)
        return total_loss

    # Optimize phi to fit the dynamic data to the canonical volume
    phi = optimize(phi, loss_fn, sparse_projections_dynamic)
    return phi

def render_dynamic_scene(canonical_theta, phi, time_t, camera_params):
    """
    Step 3: Full field spatio-temporal reconstruction.
    """
    # Use canonical volume and deformation field to render any time t
    return render_dynamic_projection(canonical_theta, phi, camera_params.angle, time_t)
```

**Edge Cases and Pitfalls:**
*   **Time as Direct Input:** Simply adding time $t$ as an input to a conventional NeRF $(x, y, z, t) \rightarrow (c, \sigma)$ does not achieve good performance [S1, unofficial].
*   **Projection Count:** Insufficient projections may violate the condition that the maximum displacement of sample points between projections is within one voxel, leading to reconstruction artifacts [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** The method demonstrates that neural radiance fields can reconstruct data modalities of interest more efficiently than conventional reconstruction methods [S1, unofficial].
*   **Strengths:** Decouples static geometry from dynamic deformation, allowing for efficient reconstruction of dynamic scenes with topologically preserving changes [S1, unofficial].
*   **Limitations:** Assumes topologically preserving temporal changes; may not handle topology changes well [S1, unofficial].
*   **When to Use:** When reconstructing dynamic scenes where a static canonical volume can be established from rich data at keyframes, and sparse data is available for intermediate times [S1, unofficial].
*   **When Not to Use:** When the scene undergoes significant topological changes over time, or when rich projection data is unavailable for static time points [S1, unofficial].

## Background
*   **Origin:** The method is described in "Neural rendering enables dynamic tomography" [S1, unofficial].
*   **History:** The work addresses the challenge of 3D reconstruction during dynamic events, moving beyond static CT. It builds on the observation that full spatio-temporal networks often struggle without extensive regularization, leading to the decoupled canonical volume approach [S1, unofficial].
*   **Usage:** Applied to real-world experiments involving lattice samples, demonstrating the reconstruction of spatio-temporal deformation [S1, unofficial].

## Sources
[S1] Neural rendering enables dynamic tomography, https://doi.org/10.48550/arxiv.2410.20558, fetched 2026-10-26 (unofficial)
