# KFD-NeRF

Also called: Kalman Filter Guided Deformation Field NeRF, Kalman Filter Guided Dynamic NeRF

## Overview
KFD-NeRF is a dynamic neural radiance field method that models the 4D radiance field as a dynamic system with temporally varying states, estimated using a Kalman filter to combine observations from input data and predictions based on local linear motion equations [S1]. It differs from sibling methods like [Gear-NeRF](13848-gear-nerf.md) and [NeRF with Time Parameter](14705-nerf-with-time-parameter.md) by explicitly separating motion estimation into an observer branch (direct observation) and a predictor branch (history-based prediction), weighted by a learnable Kalman gain, rather than relying solely on time-conditioned feature interpolation or rigid deformation priors [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** 3D point coordinates $\mathbf{x}$, viewing direction $\mathbf{d}$, and timestamp $t_i$. Training images are released in chronological order.
*   **Outputs:** Volume density $\sigma$, view-dependent color $\mathbf{c}$, and deformation field $\Delta \mathbf{x}$ (state $d_{t_i}^{\mathbf{x}}$) for the current frame [S1].

**Data Structures:**
1.  **Tri-Plane Representation:** Three sets of mutually orthogonal 2D planes encode the canonical space. Features from these planes are concatenated with raw coordinate inputs to handle coordinate shift errors [S1].
2.  **Observer MLP:** A two-layer shallow MLP takes $(\mathbf{x}, t_i)$ as input. It outputs the mean observation $y_{t_i}$ and noise-related terms $\varepsilon_{t_i}$ [S1].
3.  **Kalman Gain Network:** A linear layer $f$ takes $(\varepsilon_{t_i}, \varepsilon_{t_{i-1}}, t_i, t_{i-1})$ as input to compute the Kalman gain $K_{t_i}$ [S1].

**Numbered Steps:**
1.  **Observer Query:** For the current timestamp $t_i$, query the observer MLP with $\mathbf{x}$ and $t_i$ to obtain observation $y_{t_i}$ and noise terms $\varepsilon_{t_i}$ [S1].
2.  **Velocity Calculation:** Approximate velocity $v_{t_i}$ using deformation from the previous two frames:
    $$ v_{t_i} = (d\mathbf{x}_{t_{i-1}} - d\mathbf{x}_{t_{i-2}}) / \Delta t_i $$
    where $d\mathbf{x}_{t_{i-1}}$ and $d\mathbf{x}_{t_{i-2}}$ are the deformation states estimated at $t_{i-1}$ and $t_{i-2}$ respectively [S1].
3.  **Prediction:** Calculate the predicted deformation state $\hat{d\mathbf{x}}_{t_i}^{-}$ using the locally linear motion model:
    $$ \hat{d\mathbf{x}}_{t_i}^{-} = d\mathbf{x}_{t_{i-1}} + \Delta t_i \cdot v_{t_i} $$
    Substituting the velocity equation, this simplifies to:
    $$ \hat{d\mathbf{x}}_{t_i}^{-} = 2d\mathbf{x}_{t_{i-1}} - d\mathbf{x}_{t_{i-2}} $$
    [S1].
4.  **Kalman Gain Calculation:** Compute $K_{t_i}$ using the linear layer $f$:
    $$ K_{t_i} = f(\varepsilon_{t_i}, \varepsilon_{t_{i-1}}, t_i, t_{i-1}) $$
    [S1].
5.  **State Update:** Combine the observation $y_{t_i}$ and the prediction $\hat{d\mathbf{x}}_{t_i}^{-}$ using the Kalman gain to produce the final deformation estimation $\hat{d\mathbf{x}}_{t_i}$:
    $$ \hat{d\mathbf{x}}_{t_i} = \hat{d\mathbf{x}}_{t_i}^{-} + K_{t_i} (y_{t_i} - \hat{d\mathbf{x}}_{t_i}^{-}) $$
    [S1].
6.  **Warping:** Warp all points in real space to a time-independent canonical space according to the estimated deformation $\hat{d\mathbf{x}}_{t_i}$ [S1].
7.  **Rendering:** Query the radiance field (using the tri-plane representation and raw coordinates) at the warped canonical position to obtain $\sigma$ and $\mathbf{c}$, then render using standard volume rendering [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def kfd_nerf_deformation_step(x, t_i, t_prev, t_prev_prev, observer_mlp, kalman_gain_net, state_history):
    """
    Computes the deformation state for point x at time t_i using Kalman filtering.
    
    Args:
        x: 3D point coordinates (tensor)
        t_i: Current timestamp (float)
        t_prev: Previous timestamp (float)
        t_prev_prev: Timestamp before previous (float)
        observer_mlp: Two-layer MLP returning (observation, noise_terms)
        kalman_gain_net: Linear layer returning Kalman gain
        state_history: Dictionary or list storing d_x for previous timestamps
        
    Returns:
        d_x_t: Estimated deformation state for current frame
    """
    # Step 1: Observer for current frame
    y_t, eps_t = observer_mlp(x, t_i)
    
    # Retrieve previous states
    d_x_prev = state_history[t_prev]
    d_x_prev_prev = state_history[t_prev_prev]
    
    # Step 2: Observer for previous frames (needed for velocity and gain)
    # Note: In training, these are computed during the forward pass of previous frames.
    # Here we assume they are available or re-queried.
    y_prev, eps_prev = observer_mlp(x, t_prev)
    y_prev_prev, eps_prev_prev = observer_mlp(x, t_prev_prev)
    
    # Step 3: Calculate Velocity
    delta_t_i = t_i - t_prev_prev # Or specific delta t definition from source
    # Source formula: v_ti = (dx_{t_{i-1}} - dx_{t_{i-2}}) / delta_t_i
    # Note: The source uses dx_{t_{i-1}} and dx_{t_{i-2}}. 
    # Assuming t_prev is t_{i-1} and t_prev_prev is t_{i-2}.
    v_t = (d_x_prev - d_x_prev_prev) / (t_i - t_prev_prev)
    
    # Step 4: Prediction
    # Source formula: dx_hat_minus = 2*dx_{t_{i-1}} - dx_{t_{i-2}}
    d_x_pred = 2 * d_x_prev - d_x_prev_prev
    
    # Step 5: Kalman Gain
    # Source: K_ti = f(eps_ti, eps_{t_{i-1}}, t_i, t_{i-1})
    K_t = kalman_gain_net(eps_t, eps_prev, t_i, t_prev)
    
    # Step 6: Update
    # Source: dx_hat = dx_hat_minus + K_t * (y_t - dx_hat_minus)
    d_x_t = d_x_pred + K_t * (y_t - d_x_pred)
    
    return d_x_t
```

**Formulas:**
*   **Kalman Loss:** Minimizes error between observation and estimated deformation:
    $$ \mathcal{L}_{kf} = \frac{1}{\mathcal{N}} \sum_{\mathbf{x} \in \mathcal{N}} \| y_{t_i} - \hat{d\mathbf{x}}_{t_i} \|_2^2 $$
    [S1].
*   **Canonical Observation Loss:** Soft regularization for frame $t_0$:
    $$ \mathcal{L}_{co} = \frac{1}{\mathcal{N}} \sum_{\mathbf{x} \in \mathcal{N}} \mathbb{1}(t) d\mathbf{x} $$
    where $\mathbb{1}(t) = 1$ if $t=0$ and $0$ otherwise [S1].
*   **Total Variation Loss:** Applied to each tri-plane to encourage sparse edges:
    $$ \mathcal{L}_{tv}(\mathbf{x}) = \frac{1}{|C|} \sum_{c, i, j} (\| \mathbf{x}_c^{i,j} - \mathbf{x}_c^{i-1,j} \|_2^2 + \| \mathbf{x}_c^{i,j} - \mathbf{x}_c^{i,j-1} \|_2^2) $$
    where $c$ is a plane from collection $C$ and $i, j$ are indices on the plane’s resolution [S1].
*   **Total Loss:**
    $$ \mathcal{L} = \mathcal{L}_{image} + \mathcal{L}_{kf} + \mathcal{L}_{co} + \mathcal{L}_{prop} + \lambda_{tv} \mathcal{L}_{tv} $$
    where $\mathcal{L}_{prop}$ is the proposal sampling loss from Mip-NeRF 360 [S1].

**Parameters:**
*   **Total Variation Weight ($\lambda_{tv}$):** $1 \times 10^{-4}$ [S1].
*   **Observer MLP Depth:** 2 layers [S1].
*   **Spatial Representation:** Tri-plane [S1].

**Edge Cases and Pitfalls:**
*   **Training Order:** Images must be released in chronological order to ensure previous frames offer ample priors for current frame learning [S1].
*   **Initial vs. Later Stages:** In initial stages, predictions primarily influence the process to facilitate convergence for frames with substantial motion. In later stages, observations take precedence to recover precise and fine-grained motion details [S1].
*   **Coordinate Shift:** Tri-plane encoding introduces coordinate shift errors; concatenating raw coordinates with encoded features mitigates this [S1].

## Operating characteristics
*   **Performance (Synthetic):** PSNR 35.73 dB, SSIM 0.984, LPIPS 0.045, Average 0.0131 [S1].
*   **Performance (Real):** PSNR 28.75 dB, SSIM 0.891, LPIPS 0.115, Average 0.0389 [S1].
*   **Training Time:** 30 minutes (KFD-NeRF-S) to 3 hours (KFD-NeRF-L) on real data, compared to 48 hours for D-NeRF [S1].
*   **Memory:** 175 MB parameters for KFD-NeRF, compared to 16 MB for D-NeRF and 340 MB for KPlanes [S1].
*   **Strengths:** Faster convergence than MLP-based spatial representations; smoother deformation fields due to inductive smoothness bias of MLPs; effective handling of non-rigid motion via Kalman filtering [S1].
*   **Limitations:** Relies on local linear motion assumption, which may not capture complex non-linear dynamics perfectly; performance depends on the quality of the tri-plane representation [S1].

## Background
KFD-NeRF was introduced by Li, Niu, Zhong, Nobuhara, Nishino, and Zheng from The University of Tokyo, Kyoto University, and Shanghai Artificial Intelligence Laboratory in 2024 [S1]. It builds on the observation that dynamic radiance fields can be modeled as dynamic systems, drawing inspiration from control theory and Kalman filtering [S1]. It addresses limitations of previous methods like D-NeRF, which use deeper MLPs and suffer from slow convergence, and methods that disregard motion priors [S1].

## Sources
[S1] KFD-NeRF: Rethinking Dynamic NeRF with Kalman Filter, https://doi.org/10.1007/978-3-031-72995-9_1, fetched 2026-10-26
