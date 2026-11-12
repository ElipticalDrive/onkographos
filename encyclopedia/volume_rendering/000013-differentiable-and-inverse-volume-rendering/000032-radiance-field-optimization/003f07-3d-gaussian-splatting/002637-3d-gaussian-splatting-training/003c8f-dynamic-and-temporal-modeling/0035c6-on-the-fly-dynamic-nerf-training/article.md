# On-the-Fly Dynamic NeRF Training

Also called: OD-NeRF

## Overview
On-the-fly Dynamic NeRF Training (OD-NeRF) is a method for training dynamic Neural Radiance Fields concurrently with the acquisition of training frames, enabling the streaming of dynamic scenes as they unfold rather than reconstructing them after the fact [S1]. Unlike standard dynamic NeRFs that optimize over full video sequences, this approach minimizes rendering loss frame-by-frame using a first-order Markov assumption to maintain scalability and low latency [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A sequence of multi-view forward-facing images $C_{0:K}$ arriving over time steps $t_{0:K}$, and the radiance field estimate $F_{k-1}$ from the previous time step.
*   **Outputs:** The updated radiance field function $F_k$ at the current time step $t_k$, which can render novel views for $t_k$ [S1].

**Data Structures:**
The radiance field $F$ is represented as a neural network mapping spatial coordinates $x \in \mathbb{R}^3$ and viewing direction $d \in \mathbb{R}^3$ (and optionally time $t$) to volume density $\sigma \in \mathbb{R}$ and RGB color $c \in \mathbb{R}^3$ [S1]. The method utilizes an occupancy grid $G$ for efficient sampling, which also undergoes a transition function [S1].

**Numbered Steps:**
1.  **Initialization:** Start with an initial radiance field $F_0$ and occupancy grid $G_0$ [S1].
2.  **Observation:** Receive the set of images $C_k$ at the current time step $t_k$ [S1].
3.  **Markov Simplification:** Apply the first-order Markov assumption to simplify the probability model. Assume conditional independence such that the current radiance field $F_k$ depends only on the current observations $C_k$ and the previous state $F_{k-1}$:
    $$ F_k \perp\!\!\!\perp \{C_{0:k-1}, F_{0:k-2}\} \mid \{C_k, F_{k-1}\} $$
    This reduces the estimation target to:
    $$ F_k = \arg\max_{F_k} P(F_k \mid C_k, F_{k-1}) $$
    [S1]
4.  **Optimization:** Minimize the rendering loss to find $F_k$. The loss is the sum of squared differences between rendered colors $\hat{C}_k(r)$ and observed colors $C_k(r)$ for all rays $r$:
    $$ F_k = \arg\min_{F_k} \sum_{r \in R} \|\hat{C}_k(r) - C_k(r)\|_2^2 $$
    [S1]
5.  **Volume Rendering:** To compute $\hat{C}_k(r)$, aggregate the color of $N$ points sampled along ray $r$ with interval $\delta_i$:
    $$ \hat{C}(r) = \sum_{i=0}^{N-1} T_i (1 - \exp(-\sigma_i \delta_i)) c_i $$
    where the accumulated transmittance $T_i$ is:
    $$ T_i = \exp\left(-\sum_{j=0}^{i-1} \sigma_j \delta_j\right) $$
    [S1]
6.  **State Transition:** Update the occupancy grid $G$ using a simple transition function $P(G_k \mid G_{k-1})$ to support efficient sampling for the next step [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def on_the_fly_training_step(F_prev, G_prev, images_Ck, rays_R):
    """
    Perform one step of on-the-fly dynamic NeRF training.
    
    Args:
        F_prev: Radiance field model from time step k-1.
        G_prev: Occupancy grid from time step k-1.
        images_Ck: Tensor of images observed at time step k.
        rays_R: Set of rays corresponding to the pixels in images_Ck.
        
    Returns:
        F_curr: Updated radiance field model for time step k.
        G_curr: Updated occupancy grid for time step k.
    """
    # 1. Initialize current model as a copy or slight deformation of previous
    # The source notes a transition function P(Fk | Fk-1) but does not specify
    # the exact deformation mechanics, only that it requires minimum optimization.
    F_curr = initialize_from(F_prev) 
    
    # 2. Initialize current grid
    G_curr = transition_grid(G_prev)

    # 3. Optimization loop to minimize rendering loss
    # The source states "minimum optimization iterations per frame" but does not
    # specify the optimizer or number of steps. [G]
    for iteration in range(num_iterations):
        # Sample points along rays using G_curr for efficiency
        points, depths = sample_rays(rays_R, G_curr)
        
        # Query density and color from F_curr
        # Note: Dynamic NeRFs typically include time t in input.
        # The source defines F: (x, d) -> (sigma, c) but mentions time t in input
        # for dynamic NeRFs generally. [S1]
        sigma, color = F_curr(points, rays_R.directions, time=t_k)
        
        # Compute volume rendering for each ray
        rendered_colors = []
        for r in rays_R:
            # Get sorted points and deltas for this ray
            pts, deltas = get_sorted_samples(r, points, depths)
            sigmas, colors = get_attributes(pts, sigma, color)
            
            # Compute transmittance
            alpha = 1.0 - torch.exp(-sigmas * deltas)
            transmittance = torch.cumprod(1.0 - alpha, dim=0)
            transmittance = torch.cat([torch.ones_like(transmittance[:, :1]), transmittance[:, :-1]], dim=1)
            
            # Aggregate color
            contribution = alpha * transmittance * colors
            rendered_c = torch.sum(contribution, dim=0)
            rendered_colors.append(rendered_c)
            
        rendered_colors = torch.stack(rendered_colors)
        
        # Compute loss against observed images Ck
        observed_colors = get_pixel_colors(images_Ck, rays_R)
        loss = torch.mean((rendered_colors - observed_colors) ** 2)
        
        # Backpropagate and update F_curr
        loss.backward()
        optimizer.step()
        optimizer.zero_grad()
        
    return F_curr, G_curr
```

**Parameters:**
*   **Camera Setup:** Limited to multi-view forward-facing cameras [S1].
*   **Optimization Iterations:** The method requires "minimum optimization iterations per frame" to achieve interactive speed, but the exact number is not specified in the sources [S1].

**Edge Cases and Pitfalls:**
*   **Scalability:** Estimating $F_k$ conditioned on all previous images $C_{0:k}$ is not scalable as time $t_k$ increases. The Markov assumption is critical to mitigate this complexity growth [S1].
*   **Hardware Constraints:** The method is designed for scenarios where professional 360-degree inward-facing camera setups (dozens of cameras) are unavailable, focusing on static multi-view forward-facing setups [S1].

## Subtopics

### Neural Transient Field (NeTF)
Represents dynamic scenes using a 4D neural field with explicit time encoding, differing from OD-NeRF's streaming optimization approach. [Neural Transient Field (NeTF)](4692-neural-transient-field-netf.md)

### SWAG
Uses a spatial-temporal Gaussian mixture model for dynamic scenes, contrasting with the neural radiance field representation. [SWAG](6875-swag.md)

### Projected Color-Guided Dynamic NeRF
Incorporates color guidance for dynamic NeRF training, potentially offering different convergence properties. [Projected Color-Guided Dynamic NeRF](7523-projected-color-guided-dynamic-nerf.md)

### D-NeRF
A foundational dynamic NeRF method that reconstructs 4D space-time from full video sequences, serving as a baseline for comparison. [D-NeRF](8660-d-nerf.md)

### Dynamic Object Embedding in Static Scenes
Focuses on embedding dynamic objects within static backgrounds, a specific scene decomposition strategy. [Dynamic Object Embedding in Static Scenes](8796-dynamic-object-embedding-in-static-scenes.md)

### Dynamic 3D Gaussian Splatting (Katsumata et al.)
Applies dynamic modeling to 3D Gaussian Splatting rather than NeRFs, using different representation and rendering primitives. [Dynamic 3D Gaussian Splatting (Katsumata et al.)](9168-dynamic-3d-gaussian-splatting-katsumata-et-al.md)

## Operating characteristics
*   **Time Complexity:** The method is designed for "interactive speed" and "minimum optimization iterations per frame," significantly faster than full-sequence optimization [S1]. However, exact time complexity per frame is not given.
*   **Memory Complexity:** Uses an occupancy grid $G$ for efficient sampling, which helps manage memory during the streaming process [S1].
*   **Strengths:** Enables real-time streaming of dynamic scenes; scalable via Markov assumption; suitable for social media, VR/AR, and gaming applications [S1].
*   **Limitations:** Limited to multi-view forward-facing cameras; requires careful handling of the transition between $F_{k-1}$ and $F_k$ to avoid drift or artifacts; not suitable for monocular setups requiring camera movement or complex 360-degree setups requiring many cameras [S1].
*   **When to use:** When the goal is to stream a dynamic scene in real-time as it happens, rather than reconstructing it offline.
*   **When not to use:** When high-fidelity offline reconstruction of a past event is required, or when hardware constraints prevent multi-view forward-facing setups.

## Background
OD-NeRF was introduced to address the limitation of existing dynamic NeRFs that only reconstruct scenes after the event has been fully recorded [S1]. The work builds on the success of NeRF acceleration techniques that allow training small scenes in seconds, extending this to dynamic scenes for interactive streaming [S1]. It formulates the problem as estimating hidden states in a Hidden Markov Model (HMM), where images are observations and radiance fields are hidden states [S1].

## Sources
[S1] OD-NeRF: Efficient Training of On-the-Fly Dynamic Neural Radiance Fields, https://doi.org/10.1109/3dv66043.2025.00053, fetched 2026-10-26
