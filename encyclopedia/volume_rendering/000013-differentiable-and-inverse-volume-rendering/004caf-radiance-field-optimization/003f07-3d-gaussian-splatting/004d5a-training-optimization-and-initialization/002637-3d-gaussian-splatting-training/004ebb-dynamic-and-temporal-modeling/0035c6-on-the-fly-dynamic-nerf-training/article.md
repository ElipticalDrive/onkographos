# On-the-Fly Dynamic NeRF Training

Also called: OD-NeRF

## Overview
On-the-fly Dynamic NeRF Training (OD-NeRF) is a method for training a dynamic Neural Radiance Field concurrently with the acquisition of training frames, enabling the streaming of a dynamic scene while it unfolds [S1]. Unlike standard dynamic NeRFs that require full video sequences for offline 4D reconstruction, this approach estimates the radiance field for the current time step $t_k$ using only the images up to $C_{0:k}$ and the radiance field from the previous step $F_{0:k-1}$ [S1]. It is distinguished by its use of a first-order Markov assumption to limit computational complexity, allowing for frame-by-frame optimization with minimal iterations [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A sequence of multi-view forward-facing camera images $C_{0:K}$ captured over time $t_{0:K}$, and the initial radiance field state $F_0$ (or parameters thereof).
*   **Outputs:** A sequence of radiance field functions $F_{0:K}$, where each $F_k$ represents the scene geometry and appearance at time $t_k$.

**Data Structures:**
The radiance field $F_k$ is typically represented by a neural network (e.g., MLP) mapping spatial coordinates $\mathbf{x} \in \mathbb{R}^3$ and viewing directions $\mathbf{d} \in \mathbb{R}^3$ to volume density $\sigma \in \mathbb{R}$ and RGB color $\mathbf{c} \in \mathbb{R}^3$ [S1]. In some implementations, a deformation field $\mathbf{D}(\mathbf{x}, t)$ maps dynamic coordinates to a canonical space, or the network is expanded to a 4D variant $F': (\mathbf{x}, \mathbf{d}, t) \to (\sigma, \mathbf{c})$ [S1]. An occupancy grid $G$ may also be used for efficient sampling, with its own transition function [S1].

**Numbered Steps:**
1.  **Initialize:** Start with the radiance field state $F_0$ at time $t_0$.
2.  **Iterate for each time step $k = 1, \dots, K$:**
    a.  **Observation:** Acquire the set of images $C_k$ at time $t_k$.
    b.  **Transition/Update:** Estimate the new radiance field $F_k$ by minimizing the rendering loss between the rendered image $\hat{C}_k$ and the observed image $C_k$, conditioned on the previous field $F_{k-1}$ [S1].
    c.  **Rendering:** For each ray $r$ in the current camera views, compute the pixel color $\hat{C}_k(r)$ using volume rendering [S1].
    d.  **Optimization:** Update the parameters of $F_k$ (and potentially the deformation field or occupancy grid) to minimize the loss. The optimization starts from the state of $F_{k-1}$ to ensure smooth transitions and fast convergence [S1].
    e.  **Store:** Save $F_k$ as the current state for the next step.

**Formulas:**
The core objective at time step $k$ is to find the radiance field $F_k$ that minimizes the rendering loss [S1]:

$$ F_k = \arg\min_{F_k} \sum_{r \in R} ||\hat{C}_k(r) - C_k(r)||^2_2 $$

This is derived from the probabilistic formulation $F_k = \arg\max_{F_k} P(F_k | C_{0:k}, F_{0:k-1})$ [S1]. To make this scalable, the first-order Markov assumption is applied, assuming conditional independence $F_k \perp\!\!\!\perp \{C_{0:k-1}, F_{0:k-2}\} | \{C_k, F_{k-1}\}$ [S1]. This simplifies the estimation to [S1]:

$$ F_k = \arg\max_{F_k} P(F_k | C_k, F_{k-1}) $$

The rendered color $\hat{C}(r)$ for a ray $r$ is computed via volume rendering [S1]:

$$ \hat{C}(r) = \sum_{i=0}^{N-1} T_i (1 - \exp(-\sigma_i \delta_i)) c_i $$

where $T_i = \exp(-\sum_{j=0}^{i-1} \sigma_j \delta_j)$, $N$ is the number of points sampled along the ray, $\delta_i$ is the interval between points, and $(\sigma_i, c_i)$ are the density and color at point $i$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def on_the_fly_training(images_sequence, initial_field_params):
    """
    images_sequence: List of lists of images, where images_sequence[k] contains 
                     all views at time step k.
    initial_field_params: Parameters for the radiance field at t=0.
    """
    # Initialize the radiance field state
    current_field_params = initial_field_params
    
    # Store history if needed for analysis, though Markov assumption limits dependency
    field_history = [current_field_params]
    
    for k in range(1, len(images_sequence)):
        # Get current observations
        current_images = images_sequence[k]
        
        # Optimization loop for the current time step
        # Start optimization from the previous field state to ensure continuity
        # and minimize required iterations [S1]
        optimized_params = optimize_field(
            field_params=current_field_params,
            observations=current_images,
            loss_fn=rendering_loss
        )
        
        # Update state
        current_field_params = optimized_params
        field_history.append(current_field_params)
        
    return field_history

def rendering_loss(field_params, images):
    """
    Computes the L2 loss between rendered images and ground truth.
    """
    total_loss = 0
    for view in images:
        # Render the view using the current field parameters
        rendered_view = render_view(field_params, view.camera_params)
        # Compute L2 loss
        loss = sum((rendered_view - view.pixel_colors) ** 2)
        total_loss += loss
    return total_loss

def render_view(field_params, camera_params):
    """
    Renders a 2D image from a 3D radiance field defined by field_params.
    Uses volume rendering formula:
    C(r) = sum_{i=0}^{N-1} T_i (1 - exp(-sigma_i * delta_i)) * c_i
    """
    # 1. Generate rays for each pixel based on camera_params
    rays = generate_rays(camera_params)
    
    # 2. Sample points along each ray
    # Points are sampled based on the scene bounds and potentially an occupancy grid
    points = sample_points_along_rays(rays)
    
    # 3. Query the radiance field for density and color at each point
    # For dynamic NeRF, points may be deformed to canonical space or time is an input
    densities, colors = query_radiance_field(field_params, points)
    
    # 4. Compute transmittance and accumulated color
    # T_i = exp(-sum_{j=0}^{i-1} sigma_j * delta_j)
    # C(r) = sum_{i=0}^{N-1} T_i * (1 - exp(-sigma_i * delta_i)) * c_i
    rendered_image = volume_render(rays, densities, colors)
    
    return rendered_image
```

**Parameters:**
*   **Number of samples per ray ($N$):** Not given in the sources.
*   **Interval ($\delta$):** Not given in the sources.
*   **Optimization iterations per frame:** The method aims for "minimum optimization iterations per frame" to achieve interactive speed [S1]. Exact number not given.
*   **Camera Setup:** Limited to multi-view forward-facing cameras [S1].

**Edge Cases and Pitfalls:**
*   **Scalability:** Estimating $F_k$ conditioned on all previous images and fields is not scalable as $t_k$ increases [S1]. The Markov assumption mitigates this but may lose long-term context.
*   **Camera Motion:** The method is limited to static multi-view forward-facing cameras. Monocular or 360-degree setups are excluded due to hardware complexity or photographer movement requirements [S1].
*   **Temporal Consistency:** Rapid scene changes may require careful initialization from $F_{k-1}$ to avoid divergence or artifacts [S1].

## Subtopics

### D-NeRF
Uses a deformation field to map dynamic coordinates to a canonical space. It trains on full video sequences, unlike OD-NeRF which trains on-the-fly [S1, 8660-d-nerf.md].

### Neural Transient Field (NeTF)
Represents dynamic scenes using a 4D neural field. It typically requires offline training on complete sequences [S1, 4692-neural-transient-field-netf.md].

### SWAG
A method for dynamic scene reconstruction. It focuses on offline reconstruction rather than streaming [S1, 6875-swag.md].

### Projected Color-Guided Dynamic NeRF
Uses projected color guidance for dynamic NeRF training. It is an offline reconstruction method [S1, 7523-projected-color-guided-dynamic-nerf.md].

### Dynamic Object Embedding in Static Scenes
Separates static and dynamic components. It is designed for offline reconstruction [S1, 8796-dynamic-object-embedding-in-static-scenes.md].

### Dynamic 3D Gaussian Splatting (Katsumata et al.)
Applies dynamic modeling to 3D Gaussians. It is a differentiable rendering approach, distinct from the NeRF-based OD-NeRF [S1, 9168-dynamic-3d-gaussian-splatting-katsumata-et-al.md].

### DaReNeRF
Focuses on dynamic and realistic NeRF training. It is an offline method [S1, 13782-darenerf.md].

## Operating characteristics
*   **Time Complexity:** The method is designed for "interactive speed" by minimizing optimization iterations per frame [S1]. The complexity per step is reduced by the Markov assumption, avoiding dependency on all previous frames [S1].
*   **Memory Complexity:** By using the Markov assumption, the method avoids storing all previous images and fields for joint optimization, reducing memory requirements compared to offline methods [S1].
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Enables streaming of dynamic scenes while they happen; suitable for social media, VR/AR, and gaming applications [S1].
*   **Limitations:** Limited to multi-view forward-facing cameras; may struggle with long-term temporal consistency due to the Markov assumption; requires careful initialization to ensure smooth transitions between frames [S1].
*   **When to use:** When real-time or near-real-time streaming of a dynamic scene is required, and the scene can be captured with a static multi-view forward-facing camera setup [S1].
*   **When not to use:** When offline high-fidelity reconstruction of a past event is needed, or when the camera setup is monocular or 360-degree [S1].

## Background
On-the-fly Dynamic NeRF Training, also known as OD-NeRF, was introduced to address the limitation of existing dynamic NeRFs that only reconstruct scenes after the event has occurred [S1]. The method leverages recent advances in NeRF acceleration to enable training at interactive speeds [S1]. It formulates the problem as estimating hidden states in a Hidden Markov Model, where the radiance fields are hidden states and the images are observations [S1]. The work is motivated by applications in social media, VR/AR, and gaming industries where streaming dynamic scenes is valuable [S1].

## Sources
[S1] OD-NeRF: Efficient Training of On-the-Fly Dynamic Neural Radiance Fields, https://doi.org/10.1109/3dv66043.2025.00053, fetched 2026-10-26
