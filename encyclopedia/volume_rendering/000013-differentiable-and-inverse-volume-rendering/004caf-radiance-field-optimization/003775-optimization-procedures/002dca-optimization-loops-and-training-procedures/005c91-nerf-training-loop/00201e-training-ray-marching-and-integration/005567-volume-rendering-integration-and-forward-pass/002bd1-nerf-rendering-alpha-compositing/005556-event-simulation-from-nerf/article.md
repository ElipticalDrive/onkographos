# Event Simulation from NeRF

Also called: Log-Difference Event Model, Virtual Event Generation

## Overview
Event Simulation from NeRF generates virtual event data from a NeRF model to supervise training using real event camera observations. It solves the problem of motion blur in NeRF reconstruction by modeling the temporal integration of light during camera exposure. It differs from sibling methods like [Depth-Guided Sampling](7490-depth-guided-sampling.md) by explicitly simulating asynchronous event streams rather than static depth or color, allowing for event-driven bundle adjustment to refine camera poses and recover sharp geometry [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** NeRF model parameters $\theta$, a set of $p$ evenly sampled poses $\{T_i\}_{i=1}^p$ representing the camera trajectory during exposure, and real event data $E(x, t_i, t_{i+1})$.
*   **Outputs:** Virtual event count $\hat{E}(x, t_i, t_{i+1})$ and gradients for optimizing $\theta$ and $T_i$.

**Data Structures:**
*   **Poses:** $T_i \in SE(3)$, learnable variables representing camera poses at time $t_i$.
*   **Events:** $E(x, t_i, t_{i+1})$ is the count of real events at pixel $x$ between time $t_i$ and $t_{i+1}$.
*   **Log-Intensity:** $\mathcal{L}(T, x)$, the log-transformed grayscale intensity of the rendered image at pose $T$ and pixel $x$.

**Steps:**
1.  **Render Sharp Images:** For each sampled pose $T_i$, render the color $C(T_i, x)$ using standard volume rendering (see [NeRF Rendering (Alpha Compositing)](5124-nerf-rendering-alpha-compositing.md)).
2.  **Convert to Grayscale:** Average the RGB channels of $C(T_i, x)$ to obtain grayscale intensity $G(T_i, x)$.
3.  **Log Transform:** Compute log-intensity $\mathcal{L}(T_i, x) = \log(G(T_i, x))$.
4.  **Simulate Events:** Calculate the virtual event count $\hat{E}(x, t_i, t_{i+1})$ using the log-difference model.
5.  **Compute Loss:** Calculate the L2 loss between virtual and real events.

**Formulas:**
The virtual event count is defined as [S1]:
$$
\hat{E}(x, t_i, t_{i+1}) = \begin{cases} 
\frac{\mathcal{L}(T_{i+1}, x) - \mathcal{L}(T_i, x)}{\Theta} & \text{if } \mathcal{L}(T_{i+1}, x) \le \mathcal{L}(T_i, x) \\
\frac{\mathcal{L}(T_{i+1}, x) - \mathcal{L}(T_i, x)}{\Theta} & \text{if } \mathcal{L}(T_{i+1}, x) > \mathcal{L}(T_i, x)
\end{cases}
$$
Note: The source formula [S1] presents identical expressions for both cases, implying a linear difference scaled by $\Theta$. The threshold $\Theta$ handles the polarity in standard event models, but here it acts as a scaling factor for the log-intensity change [S1].

**Parameters:**
*   $\Theta$: Event generation threshold. Typical value: 0.3 [S1].
*   $p$: Number of sampled poses. Typical value: 5 [S1].
*   $\lambda$: Weight for event loss. Typical value: 0.005 [S1].

**Pitfalls:**
*   Event loss is calculated **only between adjacent poses** ($t_i, t_{i+1}$). Calculating between non-adjacent poses causes fluctuation and instability [S1].
*   A coarse initial pose is required for each view to start training [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def simulate_events(nerf_model, poses, theta):
    """
    Simulates virtual events from NeRF renders.
    Args:
        nerf_model: Function rendering color C(T, x) given pose T and pixel x.
        poses: List of p SE(3) poses [T_1, ..., T_p].
        theta: Event threshold (0.3).
    Returns:
        virtual_events: Tensor of shape (p-1, H, W) containing simulated event counts.
    """
    virtual_events = []
    for i in range(len(poses) - 1):
        T_curr = poses[i]
        T_next = poses[i+1]
        
        # Render sharp images for current and next pose
        # C is RGB color tensor [H, W, 3]
        C_curr = nerf_model.render(T_curr) 
        C_next = nerf_model.render(T_next)
        
        # Convert to grayscale by averaging RGB channels
        G_curr = mean(C_curr, axis=-1) # Shape [H, W]
        G_next = mean(C_next, axis=-1) # Shape [H, W]
        
        # Log transform
        # Add small epsilon to avoid log(0) if necessary [G]
        L_curr = log(G_curr + 1e-8)
        L_next = log(G_next + 1e-8)
        
        # Calculate log difference
        delta_L = L_next - L_curr
        
        # Simulate events (linear scaling by theta as per source formula)
        # Note: Source [S1] shows identical formulas for both cases.
        E_hat = delta_L / theta
        
        virtual_events.append(E_hat)
        
    return stack(virtual_events)

def compute_event_loss(virtual_events, real_events):
    """
    Computes L2 loss between virtual and real events.
    Args:
        virtual_events: Tensor [p-1, H, W]
        real_events: Tensor [p-1, H, W]
    Returns:
        loss: Scalar L2 loss.
    """
    # Sum over all pixels and pose intervals
    # p-1 intervals, m pixels
    loss = mean((virtual_events - real_events) ** 2)
    return loss
```

## Subtopics

### Event Loss
The Event Loss is the L2 loss between simulated events $\hat{E}$ and real events $E$, summed over all pixels $x \in X$ and pose intervals $i=1 \dots p-1$ [S1]. It serves as an intensity-change metric to supervise NeRF parameters and optimize adjunctive poses $T_i$ and $T_{i+1}$ via event-driven bundle adjustment [S1]. The formula is [S1]:
$$
L_{event} = \frac{1}{p-1} \sum_{i=1}^{p} \frac{1}{m} \sum_{x \in X} \| \hat{E}(x, t_i, t_{i+1}) - E(x, t_i, t_{i+1}) \|_2^2
$$
where $p$ is the number of sampled poses and $m$ is the number of pixels [S1].

## Operating characteristics
*   **Time Complexity:** Adds overhead proportional to the number of sampled poses $p$ per view, as $p$ renders are required per exposure interval. Training time is reported to be close to standard BAD-NeRF [S1].
*   **Memory Complexity:** Requires storing intermediate renders for $p$ poses per view.
*   **Performance:** Improves PSNR, SSIM, and reduces ATE compared to baselines like BAD-NeRF [S1].
*   **Strengths:** Enables joint optimization of NeRF parameters and camera poses using asynchronous event data, handling motion blur effectively.
*   **Limitations:** Requires synchronized event data and blurry images. Stability depends on adjacent pose intervals [S1].

## Background
Originated in the paper "Deblurring Neural Radiance Fields with Event-driven Bundle Adjustment" (EBAD-NeRF) by Yunshan Qi et al. [S1]. It builds on NeRF volume rendering and event-based vision models to address motion blur in dynamic scenes.

## Sources
[S1] Deblurring Neural Radiance Fields with Event-driven Bundle Adjustment, https://doi.org/10.1145/3664647.3680569, fetched 2026-10-27
