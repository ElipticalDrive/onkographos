# On-the-Fly Dynamic NeRF Training

Also called: OD-NeRF

## Overview
On-the-fly dynamic NeRF training is a method for reconstructing dynamic scenes by training a neural radiance field concurrently with the acquisition of training frames, rather than after the event concludes [S1]. Unlike existing dynamic NeRFs that require full video sequences to reconstruct 4D space-time, this approach enables streaming of the scene while it happens, requiring minimum optimization iterations per frame [S1]. It sets itself apart from siblings like [Neural Transient Field (NeTF)](4692-neural-transient-field-netf.md) and [AdaNeRF](6195-adanerf.md) by treating the radiance field evolution as a Hidden Markov Model (HMM) process, optimizing only the current state based on the previous state and current observations, thereby ensuring scalability as time progresses [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A sequence of multi-view images $C_{0:k}$ captured up to the current time step $t_k$, and the radiance field estimate $F_{k-1}$ from the previous time step $t_{k-1}$. The method assumes a first-order Markov assumption where the current radiance field $F_k$ is conditionally independent of all previous images and fields given the current image and previous field: $F_k \perp\!\!\!\perp \{C_{0:k-1}, F_{0:k-2}\} \mid \{C_k, F_{k-1}\}$ [S1].
*   **Outputs:** The updated radiance field function $F_k$ at time $t_k$, which maps spatial coordinates $\mathbf{x} \in \mathbb{R}^3$ and viewing directions $\mathbf{d} \in \mathbb{R}^3$ to volume density $\sigma \in \mathbb{R}$ and RGB color $\mathbf{c} \in \mathbb{R}^3$ [S1].

**Data Structures:**
The radiance field $F$ is typically represented by a neural network. In the context of OD-NeRF, this often includes an occupancy grid $G$ for efficient sampling, which also undergoes a transition function $P(G_k | G_{k-1})$ [S1]. The specific network architecture (e.g., MLP layers, positional encoding) is not detailed in the source notes, but the input is $(\mathbf{x}, \mathbf{d})$ and output is $(\sigma, \mathbf{c})$ [S1].

**Numbered Steps:**
1.  **Initialize:** Start with the radiance field $F_{k-1}$ and occupancy grid $G_{k-1}$ from the previous time step. If $k=0$, initialize $F_{-1}$ and $G_{-1}$ appropriately (e.g., empty or random) [G].
2.  **Transition:** Update the occupancy grid $G_k$ from $G_{k-1}$ using a transition function $P(G_k | G_{k-1})$ to maintain efficient sampling structures [S1].
3.  **Optimize:** Find the radiance field $F_k$ that minimizes the rendering loss for the current time step $t_k$, conditioned on $C_k$ and $F_{k-1}$:
    $$ F_k = \arg\min_{F_k} \sum_{\mathbf{r} \in R} ||\hat{C}_k(\mathbf{r}) - C_k(\mathbf{r})||^2_2 $$
    where $R$ is the set of rays sampled for the current frame [S1].
4.  **Render:** For each ray $\mathbf{r}$, compute the predicted color $\hat{C}_k(\mathbf{r})$ using volume rendering [S1].
5.  **Update:** Use the computed loss to update the parameters of $F_k$ (and potentially $G_k$) via gradient descent. The resulting $F_k$ is used for rendering novel views at $t_k$ and serves as $F_{k-1}$ for the next step [S1].

**Formulas:**
*   **Volume Rendering:** The predicted color $\hat{C}(\mathbf{r})$ for a ray $\mathbf{r}$ is the aggregation of $N$ points sampled along the ray:
    $$ \hat{C}(\mathbf{r}) = \sum_{i=0}^{N-1} T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i $$
    where $\sigma_i$ is the volume density, $\mathbf{c}_i$ is the RGB color, and $\delta_i$ is the interval between points [S1].
*   **Transmittance:** The transmittance $T_i$ at point $i$ is:
    $$ T_i = \exp\left(-\sum_{j=0}^{i-1} \sigma_j \delta_j\right) $$
    [S1]

**Parameters:**
*   **Number of samples $N$:** Not given in the sources.
*   **Interval $\delta$:** Not given in the sources.
*   **Learning rate:** Not given in the sources.
*   **Optimization iterations per frame:** The method requires "minimum optimization iterations" to achieve interactive speed, but the exact number is not specified [S1].

**Edge Cases and Pitfalls:**
*   **Scalability:** Estimating $F_k$ conditioned on all previous images $C_{0:k}$ is not scalable as $t_k$ increases. The first-order Markov assumption is critical to mitigate this complexity growth [S1].
*   **Camera Setup:** The method is limited to dynamic scenes captured by multi-view forward-facing cameras. It does not support monocular or 360-degree inward-facing setups in its current formulation [S1].
*   **Degenerate Input:** If the current frame $C_k$ is missing or corrupted, the update step cannot proceed correctly. The source does not specify error handling for this case [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def on_the_fly_training_step(F_prev, G_prev, C_k, rays_k, num_samples, delta):
    """
    Perform one step of on-the-fly dynamic NeRF training.
    
    Args:
        F_prev: Radiance field model from previous time step (t_{k-1}).
        G_prev: Occupancy grid from previous time step.
        C_k: Current frame images at time t_k.
        rays_k: Set of rays R for the current frame.
        num_samples: Number of points N to sample along each ray.
        delta: Interval size between samples.
        
    Returns:
        F_k: Updated radiance field model.
        G_k: Updated occupancy grid.
    """
    # Step 2: Transition occupancy grid
    # The source mentions a transition function P(G_k | G_{k-1}) but does not provide the algorithm.
    # Implementation detail: This typically involves updating grid cells based on new observations.
    G_k = transition_occupancy_grid(G_prev, C_k) 
    
    # Step 3 & 4: Optimize F_k to minimize rendering loss
    # Initialize F_k parameters, possibly copying from F_prev to leverage temporal coherence
    F_k = copy_parameters(F_prev)
    
    # Compute loss and gradients
    loss = 0.0
    for ray in rays_k:
        # Sample points along the ray
        points = sample_points(ray, num_samples, delta)
        
        # Query F_k for sigma and c at each point
        sigma_list = []
        c_list = []
        for point in points:
            sigma, c = F_k.forward(point, ray.direction)
            sigma_list.append(sigma)
            c_list.append(c)
            
        # Compute transmittance T_i
        # T_i = exp(-sum_{j=0}^{i-1} sigma_j * delta_j)
        T_list = []
        current_transmittance = 1.0
        for i in range(num_samples):
            T_list.append(current_transmittance)
            current_transmittance *= exp(-sigma_list[i] * delta)
            
        # Compute rendered color C_hat
        # C_hat = sum_{i=0}^{N-1} T_i * (1 - exp(-sigma_i * delta)) * c_i
        rendered_color = 0.0
        for i in range(num_samples):
            alpha = 1.0 - exp(-sigma_list[i] * delta)
            rendered_color += T_list[i] * alpha * c_list[i]
            
        # Get ground truth color C_k(r) for this ray
        ground_truth_color = get_ground_truth_color(C_k, ray)
        
        # Compute L2 loss
        loss += sum((rendered_color - ground_truth_color) ** 2)
        
    # Step 5: Update F_k parameters using gradient descent
    # The source does not specify the optimizer or learning rate.
    # Implementation detail: Use an optimizer like Adam.
    gradients = compute_gradients(loss, F_k.parameters())
    F_k.update_parameters(gradients)
    
    return F_k, G_k
```

## Subtopics

### Hidden Markov Model Formulation
Treats radiance fields as hidden states and images as observations, enabling efficient sequential estimation [S1].

### Occupancy Grid Transition
A mechanism to update the sampling structure $G$ between time steps to maintain efficiency [S1].

## Operating characteristics
*   **Time Complexity:** The method is designed for "interactive speed" and "minimum optimization iterations per frame" [S1]. By using the Markov assumption, it avoids the complexity of conditioning on all past frames, making it scalable over time [S1].
*   **Memory Complexity:** Depends on the size of the radiance field $F$ and occupancy grid $G$. The source does not provide specific memory usage metrics [S1].
*   **Strengths:** Enables real-time streaming of dynamic scenes; scalable as time increases due to the Markov assumption; suitable for social media, VR/AR, and gaming [S1].
*   **Limitations:** Limited to multi-view forward-facing cameras; requires a specific camera setup that may not be available to all users; performance depends on the efficiency of the transition function for $G$ and the neural network [S1].
*   **When to use:** When the goal is to stream a dynamic scene as it happens, rather than reconstructing it offline.
*   **When not to use:** When only a monocular camera is available, or when high-quality offline reconstruction of a past event is required.

## Background
*   **Origin:** Introduced in the paper "OD-NeRF: Efficient Training of On-the-Fly Dynamic Neural Radiance Fields" [S1].
*   **History:** Traditional dynamic NeRFs focus on reconstructing 4D space-time from full video sequences [S1]. This method addresses the limitation that they cannot stream the event while it is happening [S1].
*   **Applications:** Social media, VR/AR, and gaming industries [S1].

## Sources
[S1] OD-NeRF: Efficient Training of On-the-Fly Dynamic Neural Radiance Fields, https://doi.org/10.1109/3dv66043.2025.00053, fetched 2026-10-26
