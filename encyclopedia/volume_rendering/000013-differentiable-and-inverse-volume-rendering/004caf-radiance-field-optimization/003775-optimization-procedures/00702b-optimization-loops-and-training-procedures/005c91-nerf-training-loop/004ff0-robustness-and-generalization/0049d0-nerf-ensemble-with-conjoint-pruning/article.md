# NeRF Ensemble with Conjoint Pruning

Also called: CPE, Ens-FT, IPM

## Overview
NeRF Ensemble with Conjoint Pruning (CPE) improves novel view synthesis by combining two Neural Radiance Field (NeRF) models of different grid resolutions, specifically using the Direct Voxel Grid Optimization (DVGO) architecture [S1]. It addresses the high computational requirements and parameter inefficiency of single large NeRF models by demonstrating that ensembling multiple models at different scales requires fewer parameters to achieve the same image quality constraint compared to training and compressing one large model directly [S1]. Unlike vanilla ensembling, which may yield suboptimal results, CPE employs a specific fine-tuning and compression strategy to maintain a similar memory footprint and parameter count to a single model while outperforming it [S1].

## How it works
The method operates in three distinct stages: independent training, ensemble fine-tuning, and conjoint pruning [S1]. It relies on the DVGO architecture, where the scene is represented by explicit voxel grids optimized directly, avoiding the implicit MLP bottleneck of standard NeRFs [S1].

### Inputs and Outputs
*   **Inputs:** Training dataset $D_{train}$, validation dataset $D_{val}$, and the ensemble size, which is fixed at 2 models [S1].
*   **Outputs:** An ensemble of two compressed NeRF models (DVGO variants) that can be rendered jointly [S1].

### Data Structures
The ensemble consists of two separate NeRF models, denoted as $n=1$ and $n=2$ [S1]. Each model $n$ maintains its own set of parameters $w_n$ and a voxel grid [S1]. The models are trained with different grid resolutions [S1]. The source specifies two different resolutions but does not list exact numerical values [S1].

### Numbered Steps

**Stage 1: Independent Training**
1.  Initialize two NeRF models with different grid resolutions [S1].
2.  Train each model independently on the training set $D_{train}$ for a fixed number of iterations [S1].
3.  Minimize the standard NeRF loss function for each model $n$:
    $$ L(w_n) = \frac{1}{M} \sum_{m=1}^{M} \| C_m - C(w_n, x_m, \omega_m) \|^2 $$
    Where:
    *   $w_n$: Model parameters for the $n$-th NeRF [S1].
    *   $M$: Total number of training samples [S1].
    *   $C_m$: Ground truth color of the $m$-th pixel [S1].
    *   $C(w_n, x_m, \omega_m)$: Predicted color by the $n$-th NeRF for the 3D point $x_m$ and viewing direction $\omega_m$ [S1].

**Stage 2: Ensemble Fine-Tuning (Ens-FT)**
1.  Construct the ensemble by averaging the outputs of the two trained models [S1].
2.  Optimize the ensemble parameters by minimizing the difference between the ground truth and the averaged ensemble output [S1].
3.  The ensemble output $C_{avg, m}$ for a sample $m$ is:
    $$ C_{avg, m} = \frac{1}{2} \sum_{n=1}^{2} C(w_n, x_m, \omega_m) $$
    *(Note: The source defines $n=1,2$ and uses a simple average [S1]).*
4.  The loss function for this stage is:
    $$ L_{Ens-FT} = \frac{1}{M} \sum_{m=1}^{M} \| C_m - C_{avg, m} \|^2 $$
5.  Continue fine-tuning until performance on the validation set begins to drop [S1].

**Stage 3: Conjoint Pruning (CPE)**
1.  Compress the ensemble using the validation set $D_{val}$ via conjoint pruning [S1].
2.  Return the final pruned ensemble [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_nerf_ensemble(D_train, D_val, num_iterations):
    # Stage 1: Independent Training
    w1 = initialize_model(resolution="high")
    w2 = initialize_model(resolution="low")

    for _ in range(num_iterations):
        batch = sample_batch(D_train)
        loss1 = compute_loss(w1, batch)
        w1 = optimizer_step(w1, loss1)
        loss2 = compute_loss(w2, batch)
        w2 = optimizer_step(w2, loss2)

    # Stage 2: Ensemble Fine-Tuning (Ens-FT)
    while not early_stop(D_val, w1, w2):
        batch = sample_batch(D_train)
        pred1 = render_model(w1, batch)
        pred2 = render_model(w2, batch)
        pred_avg = [(p1 + p2) / 2.0 for p1, p2 in zip(pred1, pred2)]
        loss_ens = compute_loss_ensemble(batch, pred_avg)
        w1 = optimizer_step(w1, loss_ens)
        w2 = optimizer_step(w2, loss_ens)

    # Stage 3: Conjoint Pruning (CPE)
    w1, w2 = apply_conjoint_pruning(w1, w2, D_val)

    return w1, w2

def compute_loss(w, batch):
    predictions = render_model(w, batch)
    return mean_squared_error(batch.colors, predictions)

def compute_loss_ensemble(batch, pred_avg):
    return mean_squared_error(batch.colors, pred_avg)

def mean_squared_error(target, prediction):
    return sum((t - p) ** 2 for t, p in zip(target, prediction)) / len(target)

def sample_batch(dataset):
    return random.choice(dataset)

def early_stop(D_val, w1, w2):
    # Standard validation metric check [G]
    return False

def apply_conjoint_pruning(w1, w2, D_val):
    # Conjoint pruning formulation to prevent performance degradation [S1]
    # Specific algorithm details are not provided in the sources [S1]
    return w1, w2

def initialize_model(resolution):
    return {"params": [0.0] * resolution}

def render_model(w, batch):
    # Standard DVGO volumetric rendering [G]
    return [0.0] * len(batch)

def optimizer_step(w, loss):
    # Standard gradient descent update [G]
    return w
```

### Parameters and Typical Values
*   **Ensemble Size:** 2 NeRF models [S1].
*   **Grid Resolution:** Two different resolutions are used [S1].
*   **Training Iterations:** Fixed number for Stage 1 [S1].

### Edge Cases and Pitfalls
*   **Vanilla Ensembling:** Simply averaging two independently trained models without fine-tuning and pruning yields suboptimal results [S1].
*   **Pruning Degradation:** Without the proposed conjoint pruning formulation, compression may lead to significant performance degradation [S1].
*   **Precision:** The source does not specify numerical precision requirements (e.g., FP16 vs FP32) [S1].

## Operating characteristics
*   **Time Complexity:** Requires training two models independently, then fine-tuning the ensemble, which is computationally more expensive than training a single model but achieves better quality under the same parameter count constraint [S1].
*   **Memory Footprint:** The method aims to maintain a similar memory footprint and parameter count to a single model by compressing the ensemble [S1].
*   **Performance:** Beats vanilla approaches under the same parameter's cardinality constraint [S1].
*   **Strengths:** Improves novel view synthesis performance; requires fewer parameters than training and compressing one large model directly for the same quality [S1].
*   **Limitations:** Higher training complexity due to the multi-stage process [S1].

## Background
*   **Origin:** Proposed by Francesco Di Sario, Riccardo Renzulli, Enzo Tartaglione, and Marco Grangetto in 2023 for ICIAP 2023 [S1].
*   **Context:** Addresses the high computational requirements and limited scene diversity of standard NeRFs [S1]. It builds on Direct Voxel Grid Optimization (DVGO) [S1].
*   **History:** It is described as the first work proposing a joint ensembling and compression scheme for NeRF models [S1].

## Sources
[S1] Two is Better than One: Achieving High-Quality 3D Scene Modeling with a NeRF Ensemble, https://doi.org/10.1007/978-3-031-43153-1_27, fetched 2026-10-26
