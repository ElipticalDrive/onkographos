# Event Simulation from NeRF

Also called: Log-Difference Event Model, Virtual Event Generation

## Overview
Event Simulation from NeRF generates virtual event data from a NeRF model to supervise training using real event camera observations [S1]. It solves the problem of motion blur in NeRF reconstruction by modeling the temporal integration of light during camera exposure [S1]. It differs from sibling methods like [Depth-Guided Sampling](7490-depth-guided-sampling.md) by explicitly simulating asynchronous event streams rather than static depth or color, allowing for event-driven bundle adjustment to refine camera poses and recover sharp geometry [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** NeRF model parameters $\theta$, a set of $p$ evenly sampled poses $\{T_i\}_{i=1}^p$ representing the camera trajectory during exposure, and real event data $E(x, t_i, t_{i+1})$ [S1].
*   **Outputs:** Virtual event count $\hat{E}(x, t_i, t_{i+1})$ and gradients for optimizing $\theta$ and $T_i$ [S1].

**Data Structures:**
*   **Poses:** $T_i \in SE(3)$, learnable variables representing camera poses at time $t_i$ [S1].
*   **Events:** $E(x, t_i, t_{i+1})$ is the count of real events at pixel $x$ between time $t_i$ and $t_{i+1}$ [S1].
*   **Log-Intensity:** $\mathcal{L}(T, x)$, the log-transformed grayscale intensity of the rendered image at pose $T$ and pixel $x$ [S1].

**Steps:**
1.  **Render Sharp Images:** For each sampled pose $T_i$, render the color $C(T_i, x)$ using standard volume rendering (see [NeRF Rendering (Alpha Compositing)](5124-nerf-rendering-alpha-compositing.md)) [S1].
2.  **Convert to Grayscale:** Average the RGB channels of $C(T_i, x)$ to obtain grayscale intensity $G(T_i, x)$ [S1].
3.  **Log Transform:** Compute log-intensity $\mathcal{L}(T_i, x) = \log(G(T_i, x))$ [S1].
4.  **Simulate Events:** Calculate the virtual event count $\hat{E}(x, t_i, t_{i+1})$ using the log-difference model [S1].
5.  **Compute Loss:** Calculate the L2 loss between virtual and real events [S1].

**Formulas:**
The virtual event count is defined as [S1]:
$$
\hat{E}(x, t_i, t_{i+1}) = \begin{cases} 
\frac{\mathcal{L}(T_{i+1}, x) - \mathcal{L}(T_i, x)}{\Theta} & \text{if } \mathcal{L}(T_{i+1}, x) \le \mathcal{L}(T_i, x) \\
\frac{\mathcal{L}(T_{i+1}, x) - \mathcal{L}(T_i, x)}{\Theta} & \text{if } \mathcal{L}(T_{i+1}, x) > \mathcal{L}(T_i, x)
\end{cases}
$$
The source formula presents identical expressions for both cases, implying a linear difference scaled by $\Theta$ [S1]. The threshold $\Theta$ acts as a scaling factor for the log-intensity change [S1].

**Parameters:**
*   $\Theta$: Event generation threshold. Typical value: 0.3 [S1].
*   $p$: Number of sampled poses. Typical value: 5 [S1].
*   $\lambda$: Weight for event loss. Typical value: 0.005 [S1].

**Pitfalls:**
*   Event loss is calculated **only between adjacent poses** ($t_i, t_{i+1}$) [S1]. Calculating between non-adjacent poses causes fluctuation and instability [S1].
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
        virtual_events: List of tensors of shape (H, W) containing simulated event counts.
    """
    virtual_events = []
    for i in range(len(poses) - 1):
        T_curr = poses[i]
        T_next = poses[i+1]
        
        # Render sharp images for current and next pose
        C_curr = nerf_model.render(T_curr)
        C_next = nerf_model.render(T_next)
        
        # Convert to grayscale by averaging RGB channels
        G_curr = (C_curr[..., 0] + C_curr[..., 1] + C_curr[..., 2]) / 3.0
        G_next = (C_next[..., 0] + C_next[..., 1] + C_next[..., 2]) / 3.0
        
        # Log transform with epsilon guard to avoid log(0) [G]
        L_curr = log(G_curr + 1e-8)
        L_next = log(G_next + 1e-8)
        
        # Calculate log difference
        delta_L = L_next - L_curr
        
        # Simulate events (linear scaling by theta as per source formula)
        E_hat = delta_L / theta
        
        virtual_events.append(E_hat)
        
    return virtual_events

def compute_event_loss(virtual_events, real_events):
    """
    Computes L2 loss between virtual and real events.
    Args:
        virtual_events: List of tensors [H, W] for each interval.
        real_events: List of tensors [H, W] for each interval.
    Returns:
        loss: Scalar L2 loss averaged over intervals and pixels.
    """
    total_loss = 0.0
    count = 0
    for E_hat, E_real in zip(virtual_events, real_events):
        total_loss += sum((E_hat - E_real) ** 2)
        count += 1
    return total_loss / count
```

## Subtopics

### Event Loss
The Event Loss is the L2 loss between simulated events $\hat{E}$ and real events $E$, summed over all pixels $x \in X$ and pose intervals $i=1 \dots p-1$ [S1]. It serves as an intensity-change metric to supervise NeRF parameters and optimize adjunctive poses $T_i$ and $T_{i+1}$ via event-driven bundle adjustment [S1]. The formula is [S1]:
$$
L_{event} = \frac{1}{p-1} \sum_{i=1}^{p} \frac{1}{m} \sum_{x \in X} \| \hat{E}(x, t_i, t_{i+1}) - E(x, t_i, t_{i+1}) \|_2^2
$$
where $p$ is the number of sampled poses and $m$ is the number of pixels [S1].

### NeRF Rendering in Simulation
This variant uses a trained NeRF to render the scene from novel viewpoints and camera intrinsics, specifically to model a robot's camera (Logitech C920) different from the one used for training data collection [S2]. It sets itself apart by bridging simulation and reality through explicit camera calibration, and is chosen when deploying NeRF-based perception to physical hardware [S2]. The step requires matching camera intrinsics between simulation and reality by calibrating the robot’s camera, then using the obtained focal length and distortion parameters to render the NeRF [S2].

## Operating characteristics
*   **Time Complexity:** Adds overhead proportional to the number of sampled poses $p$ per view, as $p$ renders are required per exposure interval [S1]. Training time is reported to be close to standard BAD-NeRF [S1].
*   **Memory Complexity:** Requires storing intermediate renders for $p$ poses per view [G].
*   **Performance:** Improves PSNR, SSIM, and reduces ATE compared to baselines like BAD-NeRF [S1].
*   **Strengths:** Enables joint optimization of NeRF parameters and camera poses using asynchronous event data, handling motion blur effectively [S1].
*   **Limitations:** Requires synchronized event data and blurry images, and stability depends on adjacent pose intervals [S1].

## Background
Originated in the paper "Deblurring Neural Radiance Fields with Event-driven Bundle Adjustment" (EBAD-NeRF) by Yunshan Qi et al. [S1]. It builds on NeRF volume rendering and event-based vision models to address motion blur in dynamic scenes [S1].

## Sources
[S1] Deblurring Neural Radiance Fields with Event-driven Bundle Adjustment, https://doi.org/10.1145/3664647.3680569, fetched 2026-10-27
[S2] NeRF2Real: Sim2real Transfer of Vision-guided Bipedal Motion Skills using Neural Radiance Fields, https://doi.org/10.1109/icra48891.2023.10161544, fetched 2026-10-29
