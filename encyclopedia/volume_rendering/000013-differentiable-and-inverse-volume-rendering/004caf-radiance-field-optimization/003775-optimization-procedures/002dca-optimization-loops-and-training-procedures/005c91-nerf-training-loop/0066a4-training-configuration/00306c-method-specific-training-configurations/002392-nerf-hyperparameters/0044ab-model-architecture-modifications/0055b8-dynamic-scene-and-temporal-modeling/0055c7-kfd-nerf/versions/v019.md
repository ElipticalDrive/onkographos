# KFD-NeRF

Also called: Kalman Filter Guided Deformation Field NeRF, Kalman Filter Guided Dynamic NeRF

## Overview
KFD-NeRF is a dynamic neural radiance field method that models the 4D radiance field as a dynamic system with temporally varying states, estimated using a Kalman filter to combine observations from input data and predictions based on local linear motion equations [S1]. It differs from sibling methods like [Gear-NeRF](13848-gear-nerf.md) and [NeRF with Time Parameter](14705-nerf-with-time-parameter.md) by explicitly separating motion estimation into an observer branch (direct observation via a shallow MLP) and a predictor branch (history-based prediction), fusing them via a learnable Kalman gain to handle significant motion and temporal consistency [S1]. Unlike [Dynamic Appearance Particle Neural Radiance Field](15550-dynamic-appearance-particle-neural-radiance-field.md) which uses particles, KFD-NeRF uses a tri-plane spatial representation for the canonical space, allowing for faster convergence and lower memory usage compared to MLP-only backbones like [D-NeRF](14705-nerf-with-time-parameter.md) [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** 3D point coordinates $\mathbf{x}$, timestamp $t_i$, and camera parameters. Training data consists of a sequence of images released in chronological order.
*   **Outputs:** Deformation field $\Delta \mathbf{x}$ for each point, which warps real-space points to a canonical space. The canonical space is queried for density $\sigma$ and color $\mathbf{c}$ using a tri-plane representation.

**Data Structures:**
1.  **Tri-Plane Representation:** Three mutually orthogonal 2D planes encoding the canonical space. Features from these planes are concatenated with raw coordinate inputs to handle coordinate shift errors [S1].
2.  **Observer MLP:** A two-layer shallow MLP that takes $\mathbf{x}$ and $t_i$ as input and outputs the mean observation $y_{t_i}$ and noise-related terms $\varepsilon_{t_i}$ [S1].
3.  **Kalman Gain Network:** A linear layer $f$ that takes $\varepsilon_{t_i}, \varepsilon_{t_{i-1}}$ and $t_i, t_{i-1}$ as input to compute the Kalman gain $K_{t_i}$ [S1].

**Algorithm Steps:**
The deformation estimation for a point $\mathbf{x}$ at time $t_i$ follows these steps:

1.  **Observer Query:** Query the observer MLP with $\mathbf{x}$ and $t_i$ to get observation $y_{t_i}$ and noise terms $\varepsilon_{t_i}$ [S1].
2.  **Velocity Calculation:** If $t_i$ is not the first frame, approximate velocity $v_{t_i}$ using deformations from the previous two frames:
    $$
    v_{t_i} = (d\mathbf{x}_{t_{i-1}} - d\mathbf{x}_{t_{i-2}}) / \Delta t_i
    $$
    where $d\mathbf{x}_{t}$ is the deformation at time $t$ [S1].
3.  **Prediction:** Calculate the predicted deformation $\hat{d\mathbf{x}}_{t_i}^{-}$ using the local linear motion model:
    $$
    \hat{d\mathbf{x}}_{t_i}^{-} = d\mathbf{x}_{t_{i-1}} + \Delta t_i \cdot v_{t_i} = 2d\mathbf{x}_{t_{i-1}} - d\mathbf{x}_{t_{i-2}}
    $$
    [S1].
4.  **Kalman Gain Calculation:** Compute $K_{t_i}$ using the linear layer $f$:
    $$
    K_{t_i} = f(\varepsilon_{t_i}, \varepsilon_{t_{i-1}}, t_i, t_{i-1})
    $$
    [S1].
5.  **Update:** Combine observation and prediction:
    $$
    \hat{d\mathbf{x}}_{t_i} = \hat{d\mathbf{x}}_{t_i}^{-} + K_{t_i} (y_{t_i} - \hat{d\mathbf{x}}_{t_i}^{-})
    $$
    [S1].
6.  **Warping:** Warp $\mathbf{x}$ to canonical space $\mathbf{x}' = \mathbf{x} + \hat{d\mathbf{x}}_{t_i}$ [S1].
7.  **Rendering:** Query the tri-plane representation at $\mathbf{x}'$ to obtain features, which are processed to yield $\sigma$ and $\mathbf{c}$ for volume rendering [S1].

**Pseudocode:**
```python
def kfd_nerf_deformation(x, t_i, observer_mlp, kalman_gain_net, prev_deformations):
    """
    Computes the deformation for point x at time t_i using Kalman Filter guidance.
    
    Args:
        x: 3D point coordinates (tensor)
        t_i: Current timestamp (float)
        observer_mlp: Two-layer MLP returning (observation, noise_terms)
        kalman_gain_net: Linear layer returning Kalman gain
        prev_deformations: Dictionary mapping timestamp to deformation tensor
        
    Returns:
        deformation: Estimated deformation vector
    """
    # Step 1: Observer
    y_ti, eps_ti = observer_mlp(x, t_i)
    
    # Step 2 & 3: Prediction
    if t_i in prev_deformations and (t_i - 1) in prev
