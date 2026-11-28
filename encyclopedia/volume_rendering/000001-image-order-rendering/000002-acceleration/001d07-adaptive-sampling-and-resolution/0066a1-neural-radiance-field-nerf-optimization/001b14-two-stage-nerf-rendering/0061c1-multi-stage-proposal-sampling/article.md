# Multi-stage Proposal Sampling

Also called: Multi-scale Proposal Sampling, Progressive Sampling, Proposal Network Sampling (nerfstudio)

## Overview
Multi-stage proposal sampling is a hierarchical ray sampling strategy that iteratively refines ray samples using a sequence of proposal networks. [S1, unofficial] It starts with uniform sampling and subsequently uses PDF sampling weighted by the previous stage's density estimates to concentrate samples near surfaces. [S1, unofficial]

## How it works
**Inputs and Outputs:**
- **Inputs:** Ray bundle, current training step, configuration parameters (`num_proposal_network_iterations`, `num_proposal_samples_per_ray`, `num_nerf_samples_per_ray`, `anneal`), list of proposal network density functions, and previous stage weights. [G]
- **Outputs:** Final ray samples for the NeRF stage, list of weights from all proposal stages, list of ray samples from all stages. [S1, unofficial]

**Data Structures:**
- `ray_bundle`: Contains ray origins and directions. [G]
- `ray_samples`: Structured container for sample positions and frustums. [S1, unofficial]
- `weights`: Tensor of importance weights from the previous stage, used for PDF sampling. [S1, unofficial]
- `density_fns`: List of callable functions, one per proposal stage, mapping ray sample positions to density values. [S1, unofficial]

**Parameters:**
- `num_proposal_network_iterations` ($n$): Number of proposal stages. The loop runs for $i_{level} \in [0, n]$. [S1, unofficial]
- `num_proposal_samples_per_ray[i_level]`: Number of samples at proposal level $i_{level}$. [S1, unofficial]
- `num_nerf_samples_per_ray`: Number of samples at the final stage ($i_{level} == n$). [S1, unofficial]
- `anneal` ($\alpha$): Exponent for weight annealing. Default 1.0. [S1, unofficial]
- `update_sched(step)`: Function determining update frequency. [S1, unofficial]

**Algorithm Steps:**
1. **Update Check:** Determine if proposal networks need gradient updates. Update occurs if `steps_since_update > update_sched(step)` OR `step < 10`. [S1, unofficial]
2. **Loop:** Iterate $i_{level}$ from 0 to $n$ (inclusive). If $i_{level} < n$, it is a proposal stage; otherwise, it is the final NeRF stage. [S1, unofficial]
3. **PDF Sampling (for $i_{level} > 0$):** Compute annealed weights $w_{\text{annealed}} = w^{\alpha}$. Perform PDF sampling using `ray_bundle`, previous `ray_samples`, $w_{\text{annealed}}$, and `num_samples`. [S1, unofficial]
4. **Density Evaluation (if proposal stage):** If update is required, evaluate density with gradients enabled: $d = \text{density\_fns}[i_{level}](\text{positions})$. If not updated, evaluate density with gradients disabled. [S1, unofficial]
5. **Reset:** After the loop, if update occurred, reset `steps_since_update` to 0. [S1, unofficial]
6. **Return:** Final ray samples, list of all weights, list of all ray samples. [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multi_stage_proposal_sampling(
    ray_bundle,
    density_fns,
    num_proposal_network_iterations,
    num_proposal_samples_per_ray,
    num_nerf_samples_per_ray,
    anneal=1.0,
    update_sched=None,
    steps_since_update=0,
    step=0
):
    all_weights = []
    all_ray_samples = []
    current_ray_samples = None
    should_update = (steps_since_update > update_sched(step)) or (step < 10)

    for i_level in range(num_proposal_network_iterations + 1):
        is_proposal_stage = (i_level < num_proposal_network_iterations)

        if i_level == 0:
            num_samples = num_proposal_samples_per_ray[0]
            current_ray_samples = uniform_sample_ray(ray_bundle, num_samples)
        elif i_level < num_proposal_network_iterations:
            prev_weights = all_weights[-1]
            annealed_weights = prev_weights ** anneal
            num_samples = num_proposal_samples_per_ray[i_level]
            current_ray_samples = pdf_sampler(ray_bundle, all_ray_samples[-1], annealed_weights, num_samples)
        else:
            num_samples = num_nerf_samples_per_ray
            current_ray_samples = uniform_sample_ray(ray_bundle, num_samples)

        if is_proposal_stage:
            positions = current_ray_samples.frustums.get_positions()
            if should_update:
                densities = density_fns[i_level](positions)
            else:
                with torch.no_grad():
                    densities = density_fns[i_level](positions)
            weights = compute_importance_weights(densities, current_ray_samples)
            all_weights.append(weights)

        all_ray_samples.append(current_ray_samples)

    if should_update:
        steps_since_update = 0

    return current_ray_samples, all_weights, all_ray_samples

def uniform_sample_ray(ray_bundle, num_samples):
    # Standard uniform sampling along ray parameter t
    t = torch.linspace(0.0, 1.0, num_samples)
    return RaySamples(origin=ray_bundle.origin, direction=ray_bundle.direction, t=t)

def pdf_sampler(ray_bundle, prev_ray_samples, weights, num_samples):
    # Standard inverse CDF sampling
    cdf = torch.cumsum(weights, dim=-1)
    cdf = torch.cat([torch.zeros_like(cdf[:, :1]), cdf], dim=-1)
    u = torch.rand(num_samples)
    indices = torch.searchsorted(cdf, u)
    return prev_ray_samples.resample(indices)

def compute_importance_weights(densities, ray_samples):
    # Standard NeRF volume rendering weights
    deltas = ray_samples.frustums.get_deltas()
    alphas = 1.0 - torch.exp(-densities * deltas)
    transmittance = torch.cumprod(1.0 - alphas + 1e-10, dim=-1)
    weights = alphas * torch.cat([torch.ones_like(alphas[:, :1]), transmittance[:, :-1]], dim=-1)
    return weights
```

**Edge Cases and Pitfalls:**
- **Gradient Scaling Crashes:** The first 10 steps must force an update to prevent issues with gradient scaling in mixed precision training. [S1, unofficial]
- **Annealing:** If `anneal` is 1.0, no annealing occurs. [S1, unofficial]

## Subtopics

### [Combined Sampling (Occupancy Grid + Proposal Network)](1626-combined-sampling-occupancy-grid-proposal-network.md)
Combines spatial acceleration with proposal networks for faster inference.

### [VF-NeRF Rendering Pipeline](3302-vf-nerf-rendering-pipeline.md)
Uses volume features for rendering, often paired with proposal sampling.

### [Coarse-Fine Sampling](8172-coarse-fine-sampling.md)
Simple two-stage sampling without intermediate proposal networks.

### [Semantic Loss Computation](15616-semantic-loss-computation.md)
Loss function that can be applied to samples generated by proposal networks.

### [One-Step NeRF Rendering](17160-one-step-nerf-rendering.md)
Eliminates multi-stage sampling entirely for real-time rendering.

## Operating characteristics

## Background
This method is implemented in the nerfstudio framework. [S1, unofficial]

## Sources
[S1] nerfstudio-project/nerfstudio, https://github.com/nerfstudio-project/nerfstudio, fetched 2026-10-26 (unofficial)
