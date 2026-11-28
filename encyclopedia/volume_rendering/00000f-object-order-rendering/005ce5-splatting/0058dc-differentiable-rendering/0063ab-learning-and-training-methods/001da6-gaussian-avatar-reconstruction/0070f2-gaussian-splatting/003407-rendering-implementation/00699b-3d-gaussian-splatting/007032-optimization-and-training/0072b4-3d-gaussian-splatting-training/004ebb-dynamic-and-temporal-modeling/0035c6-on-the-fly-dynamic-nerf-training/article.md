# On-the-Fly Dynamic NeRF Training

Also called: OD-NeRF

## Overview
On-the-fly Dynamic NeRF Training (OD-NeRF) is a method for training dynamic Neural Radiance Fields concurrently with the acquisition of training frames, enabling the streaming of dynamic scenes as they unfold [S1]. Unlike traditional dynamic NeRFs that require full video sequences for offline 4D reconstruction, this approach estimates the radiance field for the current time step $t_k$ using only the images up to $t_k$ and the radiance field from the previous step $t_{k-1}$ [S1]. It is distinguished by its application of a first-order Markov assumption to decouple current estimation from the entire history, allowing for minimal optimization iterations per frame and interactive-speed streaming [S1].

## How it works

### Inputs and Outputs
*   **Inputs:**
    *   $C_{0:k}$: The set of multi-view images captured up to the current time step $t_k$ [S1].
    *   $F_{k-1}$: The radiance field function estimated at the previous time step $t_{k-1}$ [S1].
*   **Outputs:**
    *   $F_k$: The updated radiance field function at time $t_k$, parameterized by spatial coordinates $\mathbf{x}$ and viewing direction $\mathbf{d}$ [S1].

### Data Structures
The method relies on standard NeRF representations:
1.  **Radiance Field $F_k$:** A neural network mapping $(\mathbf{x}, \mathbf{d}) \rightarrow (\sigma, \mathbf{c})$, where $\sigma \in \mathbb{R}$ is volume density and $\mathbf{c} \in \mathbb{R}^3$ is RGB color [S1]. For dynamic scenes, this may implicitly or explicitly encode time via deformation fields or 4D expansion, but the core optimization target is the field state $F_k$ [S1].
2.  **Occupancy Grid $G_k$:** A sparse data structure used for efficient ray sampling, which is updated via a transition function from $G_{k-1}$ [S1].

### Numbered Steps

1.  **Initialize:** Start with an initial radiance field $F_0$ [S1]. The occupancy grid $G_0$ is initialized to cover the scene volume [G].
2.  **Receive Data:** At time step $k$, acquire the new set of images $C_k$ [S1].
3.  **Markov Simplification:** Apply the first-order Markov assumption to simplify the probability model [S1]. Assume conditional independence such that the current radiance field $F_k$ is independent of all previous images $\{C_{0:k-1}\}$ and fields $\{F_{0:k-2}\}$ given the current images $C_k$ and previous field $F_{k-1}$ [S1]:
    $$ F_k \perp\!\!\!\perp \{C_{0:k-1}, F_{0:k-2}\} \mid \{C_k, F_{k-1}\} $$
    This reduces the estimation objective to:
    $$ F_k = \arg\max_{F_k} P(F_k \mid C_k, F_{k-1}) $$
    [S1]
4.  **Define Loss:** Minimize the rendering loss between the rendered image $\hat{C}_k(r)$ and the ground truth image $C_k(r)$ for all rays $r \in R$ [S1]:
    $$ F_k = \arg\min_{F_k} \sum_{r \in R} ||\hat{C}_k(r) - C_k(r)||^2_2 $$
    [S1]
5.  **Volume Rendering:** For each ray $r$, compute the rendered color $\hat{C}_k(r)$ by aggregating $N$ points along the ray with intervals $\delta_i$ [S1]:
    $$ \hat{C}_k(r) = \sum_{i=0}^{N-1} T_i (1 - \exp(-\sigma_i \delta_i)) c_i $$
    where the transmittance $T_i$ is:
    $$ T_i = \exp\left(-\sum_{j=0}^{i-1} \sigma_j \delta_j\right) $$
    Here, $\sigma_i$ and $c_i$ are the density and color queried from $F_k$ at the $i$-th point on the ray [S1].
6.  **Optimization:** Update the parameters of $F_k$ (and potentially $G_k$) using gradient descent to minimize the loss defined in Step 4 [S1]. The optimization starts from the state $F_{k-1}$ to ensure continuity and minimize the number of iterations required [S1].
7.  **Transition:** Update the occupancy grid $G_k$ from $G_{k-1}$ using a simple transition function to maintain efficient sampling [S1].
8.  **Stream:** Render novel views using $F_k$ at time $t_k$ [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def on_the_fly_training_step(F_prev, images_Ck, cameras_k, num_iterations_per_frame, delta):
    # Initialize current field parameters from previous step
    F_curr = copy.deepcopy(F_prev)
    
    # Volume rendering function matching the formula exactly using forward accumulation
    def volume_render(points, directions):
        # points: (N, 3), directions: (N, 3)
        sigma, color = F_curr(points, directions)
        # sigma: (N,), color: (N, 3)
        rendered = np.zeros(3)
        T = 1.0
        for i in range(len(sigma)):
            alpha = 1.0 - np.exp(-sigma[i] * delta)
            rendered += T * alpha * color[i]
            T *= (1.0 - alpha)
        return rendered

    # Loss computation over all cameras and pixels
    def compute_loss():
        total_loss = 0.0
        for cam in cameras_k:
            for u, v in cam.pixel_coordinates:
                ray_origin = cam.position
                ray_direction = normalize(cam.focal_matrix @ np.array([u, v, 1.0]) - ray_origin)
                points = ray_origin + np.array([ray_direction * (i * delta) for i in range(N)])
                rendered = volume_render(points, ray_direction)
                gt = cam.image[u, v]
                total_loss += np.sum((rendered - gt) ** 2)
        return total_loss

    # Optimization loop
    for _ in range(num_iterations_per_frame):
        loss = compute_loss()
        gradients = compute_gradient(loss, F_curr.weights)
        F_curr.weights -= learning_rate * gradients
        
    return F_curr
```

### Parameters
*   **Number of optimization iterations per frame:** The exact number is not given in the sources [S1].
*   **Number of samples $N$:** Not given in the sources.
*   **Interval $\delta$:** Not given in the sources.
*   **Camera Setup:** The method is limited to dynamic scenes captured by multi-view forward-facing cameras [S1].
*   **Memory Complexity:** Memory usage is dominated by the radiance field $F_k$ and the occupancy grid $G_k$ [S1].

### Edge Cases and Pitfalls
*   **Scalability:** Estimating $F_k$ conditioned on all previous images $C_{0:k}$ is not scalable as time $t_k$ increases. The Markov assumption is critical to mitigate this complexity growth [S1].
*   **Camera Setup:** The method is limited to dynamic scenes captured by multi-view forward-facing cameras. Monocular setups are excluded due to the need for camera movement, which is cumbersome for prolonged streaming [S1]. 360-degree inward-facing setups are also excluded due to hardware complexity [S1].
*   **Continuity:** The optimization must start from $F_{k-1}$ to ensure smooth transitions and minimize optimization steps [S1].
*   **Precision:** The volume rendering loop must use forward transmittance accumulation to avoid numerical underflow or off-by-one errors when rays start inside volumes or use fine step sizes [G].

## Operating characteristics

### Time and Memory Complexity
*   **Time Complexity:** The method aims for interactive speed by minimizing optimization iterations per frame [S1]. It avoids the $O(K)$ complexity of processing all $K$ frames jointly, instead operating in $O(1)$ per frame with respect to the history length, thanks to the Markov assumption [S1].
*   **Memory Complexity:** Memory usage is dominated by the radiance field $F_k$ and the occupancy grid $G_k$ [S1]. It does not need to store the full history of radiance fields or images, only the current state and the immediate previous state [S1].

### Measured Performance
*   Not given in the sources.

### Strengths and Limitations
*   **Strengths:**
    *   Enables real-time streaming of dynamic scenes as they happen [S1].
    *   Scalable to long-duration events by avoiding history accumulation [S1].
    *   Compatible with NeRF acceleration techniques (e.g., occupancy grids) [S1].
*   **Limitations:**
    *   Limited to multi-view forward-facing camera setups [S1].
    *   Requires careful initialization and transition between time steps to maintain quality with minimal iterations [S1].

### When to Use
*   Use when the goal is to stream a dynamic scene in real-time or near-real-time, and the camera setup allows for multi-view forward-facing capture [S1].

### When Not to Use
*   Do not use with monocular or complex multi-view inward-facing setups [S1].

## Background

### Origin
The method was introduced in the paper "OD-NeRF: Efficient Training of On-the-Fly Dynamic Neural Radiance Fields" [S1].

### History and Development
Traditional dynamic NeRFs (e.g., D-NeRF, Nerfies) focus on reconstructing 4D space-time from full video sequences [S1]. OD-NeRF addresses the limitation that these methods cannot stream the event while it is happening [S1]. It builds on the success of NeRF acceleration techniques that allow training small scenes in seconds, extending this to dynamic scenes on-the-fly [S1].

### Applications
The method enables applications in social media, VR/AR, and gaming industries where real-time streaming of dynamic 3D content is valuable [S1].

## Sources

[S1] OD-NeRF: Efficient Training of On-the-Fly Dynamic Neural Radiance Fields, https://doi.org/10.1109/3dv66043.2025.00053, fetched 2026-10-26
