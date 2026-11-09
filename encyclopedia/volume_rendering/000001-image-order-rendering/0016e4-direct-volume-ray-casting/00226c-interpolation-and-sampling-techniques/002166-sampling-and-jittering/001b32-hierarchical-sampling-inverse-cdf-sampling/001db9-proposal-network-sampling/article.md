# Proposal Network Sampling

Also called: Proposal Network, Proposal Network Sampler, PDF Approaches

## Overview
Proposal Network Sampling is a method that uses a coarse Multi-Layer Perceptron (MLP) to directly estimate the Probability Density Function (PDF) along a ray, enabling importance sampling that concentrates samples on high-contribution areas (surfaces) while spreading them in empty or occluded regions [S1]. Unlike spatial skipping methods (e.g., [RangeFineSampler](3141-rangefinesampler.md)) which skip empty space entirely, proposal networks always spread samples throughout the space but with varying density, providing more accurate transmittance estimation at the cost of higher sample counts in empty regions [S1]. It builds on [Hierarchical Sampling (Inverse CDF Sampling)](973-hierarchical-sampling-inverse-cdf-sampling.md) by providing the discrete probability distribution via a learned neural estimator rather than a static grid or uniform distribution [G].

## How it works
**Inputs:** Ray parameters (origin, direction, near/far planes), coarse MLP weights (trained parameters) [S1, S2].
**Outputs:** Sample locations $t_i$ along the ray, sampled according to the estimated PDF $p(t_i)$ [S1].

**Data Structures:**
*   **Coarse MLP (Proposal Network):** A standard MLP taking ray parameters (e.g., position, direction) as input and outputting density $\sigma(t)$ at queried locations. The size is typically comparable to the shading network [S2].
*   **Transmittance Estimator:** Implicitly maintained via the density outputs and the exponential transmittance formula [S1].

**Steps:**
1.  **Training Phase:** Train the coarse MLP using volumetric rendering loss to output densities $\sigma(t_i)$ [S1].
2.  **PDF Construction:** Construct a piece-wise constant PDF from the discrete density outputs:
    $$ p(t_i) = \sigma(t_i) \exp(-\sigma(t_i) dt) $$
    where $dt$ is the distance between samples [S1].
3.  **Transmittance Estimation:** Compute the piece-wise linear transmittance estimator $T(t_i)$ for importance sampling:
    $$ T(t_i) = \exp\left(-\sum_{j=1}^{i-1} \sigma(t_j) dt\right) $$
    [S1].
4.  **Optimization:** Update the transmittance estimator (and thus the MLP weights) through gradient descent using the rendering loss [S1].
5.  **Inference/Sampling:** Use the trained PDF $p(t_i)$ to sample new locations $t_i$ along the ray using inverse CDF sampling (as described in [Hierarchical Sampling (Inverse CDF Sampling)](973-hierarchical-sampling-inverse-cdf-sampling.md)) [S1].

**Parameters:**
*   **Network Size:** The oracle/proposal network often uses an MLP of the same size as the shading network to generate consistent depth estimates [S2].
*   **Update Schedule:** In some implementations (e.g., GANeRF), proposal network updates are interpolated based on training steps to balance quality and speed [S3, unofficial].
    *   `proposal_warmup`: Number of steps for interpolation [S3, unofficial].
    *   `proposal_update_every`: Maximum frequency (in steps) for updates [S3, unofficial].
*   **Sharing:** A single proposal network can be shared across all proposal iterations via `use_same_proposal_network` [S3, unofficial].

**Edge Cases and Pitfalls:**
*   **Depth Discontinuities:** Accurate estimates around depth discontinuities are difficult, leading to visual artifacts if the oracle predicts exact geometric structures [S2]. Mitigation: Allow the oracle to return the same result for all rays around discontinuities (sampling both foreground and background) to make the task easier to learn [S2].
*   **Aliasing:** Introduces aliasing effects due to the piece-wise linearity assumption for estimating transmittance along the ray [S1].
*   **No Skipping:** Samples are always spread out throughout the entire space without any skipping, which can be less efficient than spatial skipping in highly empty scenes [S1].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

class ProposalNetwork(nn.Module):
    """Coarse MLP estimating density sigma(t) along a ray."""
    def __init__(self, input_dim=6, hidden_dim=256, num_layers=4):
        super().__init__()
        layers = []
        for i in range(num_layers):
            layers.append(nn.Linear(input_dim if i == 0 else hidden_dim, hidden_dim))
            layers.append(nn.ReLU())
        layers.append(nn.Linear(hidden_dim, 1))
        self.network = nn.Sequential(*layers)

    def forward(self, ray_origins, ray_directions, t_vals):
        """Query density at sample points along rays.
        Args:
            ray_origins: (N, 3)
            ray_directions: (N, 3)
            t_vals: (N, K) distances along ray
        Returns:
            sigmas: (N, K) estimated density
        """
        pts = ray_origins[:, None, :] + t_vals[:, :, None] * ray_directions[:, None, :]
        pts_flat = pts.reshape(-1, 3)
        ray_dirs_flat = ray_directions[:, None, :].expand(-1, t_vals.shape[1], -1).reshape(-1, 3)
        input_vec = torch.cat([pts_flat, ray_dirs_flat], dim=-1)
        sigma = self.network(input_vec).squeeze(-1)
        return sigma.reshape(ray_origins.shape[0], t_vals.shape[1])

def sample_from_proposal_network(proposal_net, ray_origins, ray_directions, num_samples, near_plane, far_plane):
    """Sample locations along the ray using the proposal network's estimated PDF."""
    # 1. Initialize initial samples (e.g., uniform)
    t_vals = torch.linspace(near_plane, far_plane, num_samples + 1).to(ray_origins.device)
    t_mids = 0.5 * (t_vals[:-1] + t_vals[1:])
    
    # 2. Query proposal network for densities
    sigmas = proposal_net(ray_origins, ray_directions, t_mids)
    
    # 3. Compute dt and PDF
    dt = t_vals[1:] - t_vals[:-1]
    # PDF: p(t_i) = sigma(t_i) * exp(-sigma(t_i) * dt) [S1]
    pdf = sigmas * torch.exp(-sigmas * dt)
    
    # Normalize PDF to sum to 1 over the ray
    pdf_sum = pdf.sum(dim=-1, keepdim=True)
    pdf_normalized = pdf / (pdf_sum + 1e-10)
    
    # 4. Inverse CDF sampling
    cdf = torch.cumsum(pdf_normalized, dim=-1)
    cdf = torch.cat([torch.zeros_like(cdf[:, :1]), cdf], dim=-1)
    
    # Generate uniform random numbers
    u = torch.rand_like(cdf[:, :-1])
    
    # Find bin indices
    indices = torch.searchsorted(cdf[:, :-1], u, right=True)
    
    # Interpolate t within the bin
    t_low = t_mids[torch.arange(u.shape[0]), indices]
    t_high = t_mids[torch.arange(u.shape[0]), indices + 1]
    cdf_low = cdf[torch.arange(u.shape[0]), indices]
    cdf_high = cdf[torch.arange(u.shape[0]), indices + 1]
    
    t_sample = t_low + (u - cdf_low) / (cdf_high - cdf_low + 1e-10) * (t_high - t_low)
    return t_sample
```

## Subtopics

### Histogram Dilation
A technique to dilate proposal histograms to reduce aliasing artifacts caused by the proposal MLP being supervised only on rays corresponding to input pixels [S4]. It widens the intervals of the proposal MLP to counteract rotational aliasing [S4].
*   **Method:** Compute probability density $\hat{p}_i = \hat{w}_i / (\hat{s}_{i+1} - \hat{s}_i)$ from histogram $(\hat{s}, \hat{w})$ [S4]. Dilate by computing $\max_{s-\epsilon \leq s' < s+\epsilon} \hat{\boldsymbol{p}}_{\hat{\boldsymbol{s}}}(s')$ [S4]. Construct new intervals with endpoints $\text{sort}(\hat{s} \cup \hat{s} - \epsilon \cup \hat{s} + \epsilon)$ [S4]. Compute max of all intervals in expanded set [S4]. Convert dilated $\hat{p}$ back to histogram by multiplying by interval size [S4]. Normalize histogram to sum to 1 [S4].
*   **Parameters:** Dilation factor $\epsilon_k = \frac{a}{\prod_{k'=1}^{k-1} n_{k'}} + b$ where $n_{k'}$ is the sample count at level $k'$ [S4]. Hyperparameters $a = 0.5$ and $b = 0.0025$ [S4].

### Chunked Proposal Network Rendering
A chunked rendering pipeline that uses a PropNetEstimator to sample from a sequence of proposal networks, then composites results using the main radiance field [S5, unofficial].
*   **Method:** Define `prop_sigma_fn` to query proposal networks at midpoints [S5, unofficial]. Define `rgb_sigma_fn` to query the radiance field for RGB and sigma at midpoints [S5, unofficial]. Loop over rays in chunks [S5, unofficial]. Call `estimator.sampling` with a list of proposal sigma functions, `prop_samples` per network, total `num_samples`, and `sampling_type` [S5, unofficial]. Call rendering with `t_starts`, `t_ends`, `ray_indices=None`, `n_rays=None`, and `rgb_sigma_fn` to composite [S5, unofficial]. Collate chunk results using `torch.cat` for tensors and reshape to image dimensions [S5, unofficial].

### Learning Sample Importance
A mechanism to predict the importance of each proposal sample to reduce computational burden by querying the fine network only on important samples [S6, unofficial].
*   **Method:** The proposer predicts importance scores by regressing another set of $N_c + N_f$ scalars using the same mechanism as proposal generation (e.g., a fully connected head on the 'ray representation' vector) [S6, unofficial]. Training uses the weight $w_i$ that each fine sample contributes to the final rendering as the target signal [S6, unofficial]. Threshold these weights at 0.03 to define positive (important) and negative (unimportant) samples [S6, unofficial]. Train the predictor using balanced logistic regression loss [S6, unofficial].
*   **Parameters:** Importance threshold: 0.03 [S6, unofficial].

### Deep Learning Path Guiding
A method where a deep neural network models the probability density function of samples to guide the ray's outgoing direction [S7]. It is independent from scenes and local point parameters (e.g., including the textured BRDF) [S7]. However, the method is biased and sampling is very expensive [S7].

## Operating characteristics
*   **Time Complexity:** Depends on the number of samples and the cost of querying the proposal MLP [S1].
*   **Memory Complexity:** Requires storing the proposal MLP weights [S1].
*   **Performance:** Provides more accurate transmittance estimation, concentrating samples on surfaces and spreading them in empty/occluded regions [S1]. However, it introduces aliasing effects due to piece-wise linearity assumptions [S1].
*   **Strengths:** Adaptive sampling based on learned scene structure; effective for complex scenes with varying density [S1].
*   **Limitations:** Samples are always spread out without skipping, which can be inefficient in highly empty scenes compared to spatial skipping methods [S1]. Difficult to learn accurate estimates around depth discontinuities [S2].
*   **When to use:** When accurate transmittance estimation is critical and the scene has complex density variations that uniform or grid-based sampling cannot efficiently capture [S1].
*   **When not to use:** When memory or compute is severely constrained and the scene is mostly empty (spatial skipping may be better) [S1].

## Background
Proposal Network Sampling originated from the need to improve importance sampling in Neural Radiance Fields (NeRFs) [S1]. It was introduced in Mip-NeRF 360 [S1], which proposed using a smaller "Proposal Network" MLP to speed up PDF construction while maintaining accuracy. The method is part of a broader class of "PDF Approaches" that directly estimate the PDF along the ray, as opposed to "Spatial Skipping" approaches like Occupancy Grids [S1]. It builds on the concept of transmittance-based importance sampling, where the transmittance estimator is updated through gradient descent during optimization [S1]. Variants include using depth oracle networks for compact representations [S2] and learning sample importance to reduce fine network queries [S6, unofficial].

## Sources
[S1] NerfAcc: Efficient Sampling Accelerates NeRFs, https://doi.org/10.1109/iccv51070.2023.01699, fetched 2026-10-26
[S2] DONeRF: Towards Real‐Time Rendering of Compact Neural Radiance Fields using Depth Oracle Networks, https://doi.org/10.1111/cgf.14340, fetched 2026-10-26
[S3] barbararoessle/ganerf, https://github.com/barbararoessle/ganerf, fetched 2026-10-27 (unofficial)
[S4] Mip-NeRF 360: Unbounded Anti-Aliased Neural Radiance Fields, https://doi.org/10.1109/cvpr52688.2022.00539, fetched 2026-10-26
[S5] Yosshi999/faster-occgrid, https://github.com/Yosshi999/faster-occgrid, fetched 2026-10-24 (unofficial)
[S6] NeRF in detail: Learning to sample for view synthesis, https://doi.org/10.48550/arxiv.2106.05264, fetched 2026-10-27 (unofficial)
[S7] A survey on rendering homogeneous participating media, https://doi.org/10.1007/s41095-021-0249-1, fetched 2026-10-26
