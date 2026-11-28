# TensoRF with Density-Shift Fix

Also called: Corrected TensoRF, Corrected TensoRF Density Parameterization, Density shift correction, Novelty I, TensoRF density shift correction

## Overview
TensoRF with Density-Shift Fix is a parameter correction for the TensoRF [13296-tensorf-decomposition-models.md] density parameterization that prevents the density field from collapsing to zero during training [S1, unofficial]. It replaces the default configuration ($\Delta = -10, s = 25$) which causes degenerate attenuation geometry and zero gradient flow, with a corrected configuration ($\Delta = 0, s = 1$) that stabilizes volumetric reconstruction and thresholding [S1, unofficial]. Unlike the default TensoRF which may achieve high photometric fidelity (PSNR 32.0 dB) but unstable geometry, this fix yields stable volumetric errors across adjacent percentiles [S1, unofficial].

## How it works
**Inputs:**
- TensoRF model parameters $\theta$ (including feature planes/lines $P_i, L_i$ and MLP weights) [S1, unofficial].
- Camera poses and intrinsics [G].
- Target images $I$ [S1, unofficial].

**Outputs:**
- Corrected volumetric density field $\sigma(x)$ [S1, unofficial].
- Optimized model parameters $\theta$ [S1, unofficial].

**Data Structures:**
- **TensoRF Model:** Uses Vector-Matrix (VM) decomposition [S1, unofficial].
- Feature planes $P_i$ (size: $48 \times 48 \times 48$ per axis, 27-dim features) [S1, unofficial].
- Feature lines $L_i$ (size: $16$ per axis, VM rank-16) [S1, unofficial].
- MLP: $2 \times 128$ hidden units, ReLU activation, 6 frequencies [S1, unofficial].
- Scene AABB: $x \in [-2.5, 3.5], y \in [-1.5, 1.5], z \in [-2.0, 1.5]$ [S1, unofficial].
- Voxel grid: $64^3$ voxels ($\approx 3.75$ mm per voxel) [S1, unofficial].

**Parameters:**
- $\Delta$ (density shift): Corrected value is $0$ (Default is $-10$) [S1, unofficial].
- $s$ (distance scale): Corrected value is $1$ (Default is $25$) [S1, unofficial].
- $\lambda_1$ (sparsity weight): $1 \times 10^{-4}$ [S1, unofficial].
- $\lambda_2$ (TV regularization weight): $1 \times 10^{-5}$ [S1, unofficial].

**Formulas:**
The corrected density $\sigma(x)$ at position $x$ is computed as [S1, unofficial]:
$$ \sigma(x) = \text{ReLU}(f_\theta(x) + \Delta) \cdot s $$
where $f_\theta(x)$ is the raw output of the TensoRF density network. With the correction ($\Delta=0, s=1$):
$$ \sigma(x) = \text{ReLU}(f_\theta(x)) $$
The total loss function is defined as [S1, unofficial]:
$$ \mathcal{L} = \mathcal{L}_{\text{photo}} + \lambda_1 \mathcal{L}_{\text{sparse}} + \lambda_2 \mathcal{L}_{\text{TV}} $$
where $\mathcal{L}_{\text{photo}} = \text{MSE}(\hat{I}, I)$, $\mathcal{L}_{\text{sparse}}$ is an $\ell_1$ penalty on density, and $\mathcal{L}_{\text{TV}}$ is total variation regularization on density planes [S1, unofficial]:
$$ \mathcal{L}_{\text{TV}} = \sum_i \left( \|\Delta_h P_i\|_1 + \|\Delta_v P_i\|_1 \right) $$
where $\Delta_h, \Delta_v$ are horizontal and vertical finite differences [S1, unofficial].

**Steps:**
1. Initialize TensoRF model parameters $\theta$ using Kaiming initialization. This yields $f_\theta(x) \in [-c, c]$ where $c \approx 1$ [S1, unofficial].
2. Set density shift $\Delta = 0$ and scale $s = 1$ in the TensoRF configuration [S1, unofficial].
3. Render views using the corrected density parameterization $\sigma(x) = \text{ReLU}(f_\theta(x))$ [S1, unofficial].
4. Compute photometric loss $\mathcal{L}_{\text{photo}}$, sparsity loss $\mathcal{L}_{\text{sparse}}$, and total variation loss $\mathcal{L}_{\text{TV}}$ [S1, unofficial].
5. Update parameters $\theta$ via gradient descent to minimize $\mathcal{L}$ [S1, unofficial].
6. After training, perform a threshold sweep on the resulting density field for volumetric extraction (e.g., Marching Cubes) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def relu(x):
    return max(x, 0.0)

def mse(a, b):
    return sum((ai - bi) ** 2 for ai, bi in zip(a, b)) / len(a)

def l1_norm(x):
    return sum(abs(xi) for xi in x)

def compute_total_variation(planes):
    tv = 0.0
    for plane in planes:
        for i in range(len(plane) - 1):
            for j in range(len(plane[0]) - 1):
                tv += abs(plane[i + 1][j] - plane[i][j]) + abs(plane[i][j + 1] - plane[i][j])
    return tv

def psnr(a, b, max_val=1.0):
    m = mse(a, b)
    return 10.0 * (max_val ** 2 / m) if m > 0 else float('inf')

def corrected_tensorf_density(f_theta_x, delta=0.0, s=1.0):
    return [relu(f + delta) * s for f in f_theta_x]

def train_corrected_tensorf(model_params, images, poses, lambda1=1e-4, lambda2=1e-5, iterations=3000):
    delta, s = 0.0, 1.0
    for i in range(iterations):
        rendered = forward_pass(model_params, poses)  # [G] Standard volume rendering forward pass
        l_photo = mse(rendered, images)
        density = corrected_tensorf_density(model_params.f_theta, delta, s)
        l_sparse = l1_norm(density)
        l_tv = compute_total_variation(model_params.density_planes)
        loss = l_photo + lambda1 * l_sparse + lambda2 * l_tv
        update_params(model_params, loss)  # [G] Standard gradient descent step
    return model_params
```

**Edge Cases and Pitfalls:**
- **Dead-Zone Failure:** Using the default $\Delta = -10$ with Kaiming initialization ($f_\theta(x) \in [-1, 1]$) results in $f_\theta(x) + \Delta \in [-11, -9]$ [S1, unofficial]. The ReLU output is zero, gradients are zero, and density remains zero throughout training [S1, unofficial].
- **Photometric vs. Geometric Fidelity:** The default parameterization may achieve high PSNR (32.0 dB) because the appearance MLP can still update, but the density field remains uniformly zero, leading to unstable marching cubes results (e.g., 204% error at p=97) [S1, unofficial].
- **Threshold Sensitivity:** The corrected version stabilizes errors across adjacent percentiles (e.g., 13.8% at p=99 and 14.3% at p=97 on patient LIDC-IDRI-0001), confirming that the parameterization is a prerequisite for downstream reconstruction quality [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** On patient LIDC-IDRI-0001, the corrected method achieves 13.8% error at one percentile and 14.3% at an adjacent percentile [S1, unofficial]. The default method achieves 8.0% error at p=99 but 204% at p=97 [S1, unofficial].
- **Strengths:** Restores gradient flow immediately; stabilizes volumetric reconstruction; robust to variation in initialization range $c$ [S1, unofficial].
- **Limitations:** May reduce photometric fidelity (PSNR) compared to the default configuration which can "learn" to reproduce pixel intensities via the appearance branch despite zero density [S1, unofficial].
- **When to Use:** When stable volumetric extraction (e.g., for medical nodule volumetry) is required, and the default TensoRF density collapse is observed [S1, unofficial].
- **When Not to Use:** When only photometric rendering quality is needed and geometric consistency is not critical [S1, unofficial].

## Background
- **Origin:** Identified in "Sparse-View Lung Nodule Volumetry from Digitally Reconstructed Radiographs via AReT: Anatomy-Regularized TensoRF" [S1, unofficial].
- **History:** The default TensoRF parameterization ($\Delta = -10, s = 25$) was found to cause a critical failure mode where density collapses to zero due to the interaction with Kaiming initialization [S1, unofficial]. The fix ($\Delta = 0, s = 1$) was proposed as "Novelty I" in this work [S1, unofficial].
- **Usage:** Applied in medical imaging, specifically for lung nodule volumetry from sparse-view X-ray projections (LIDC-IDRI dataset) [S1, unofficial].

## Sources
[S1] Sparse-View Lung Nodule Volumetry from Digitally Reconstructed Radiographs via AReT: Anatomy-Regularized TensoRF, https://doi.org/10.48550/arxiv.2606.02639, fetched 2026-10-26 (unofficial)
