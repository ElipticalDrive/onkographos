# NeRF Ensemble with Conjoint Pruning

Also called: CPE, Ens-FT, IPM

## Overview
NeRF Ensemble with Conjoint Pruning is a training strategy that improves novel view synthesis quality by combining two Neural Radiance Field (NeRF) models of different grid resolutions, specifically using the Direct Voxel Grid Optimization (DVGO) architecture [S1]. It addresses the high computational requirements and optimization difficulties of fine-grained voxel grids by achieving superior performance compared to single models while maintaining a similar parameter count and memory footprint [S1]. Unlike vanilla ensembling, which may yield suboptimal results, this method employs a specific three-stage process involving independent training, fine-tuning, and conjoint pruning to prevent performance degradation [S1].

## How it works
The method operates on an ensemble of two NeRF models, denoted as $n=1, 2$, where each model $n$ has its own set of parameters $w_n$ and voxel grid [S1]. The models are trained with different grid resolutions [S1]. The process consists of three distinct stages [S1].

### Inputs and Outputs
- **Inputs**: Training set $D_{train}$, validation set $D_{val}$, two initial NeRF models with different grid resolutions.
- **Outputs**: An ensemble of two compressed NeRF models.

### Data Structures
Each NeRF model $n$ maintains:
- Parameters $w_n$: Includes the voxel grid values and MLP weights (if applicable, though DVGO is primarily grid-based).
- Voxel Grid: A 3D grid of density and color values. The two models have different grid resolutions [S1].

### Steps

**Stage 1: Independent Training**
Train the two NeRF models independently on the training set $D_{train}$ [S1]. For each model, minimize the standard NeRF loss function for a fixed number of iterations [S1]:

$$
L(w) = \frac{1}{M} \sum_{m=1}^{M} \| C_m - C(w, x_m, \omega_m) \|^2
$$

Where:
- $w$: Model parameters.
- $C_m$: Ground truth color of the $m$-th pixel.
- $C(w, x_m, \omega_m)$: Predicted color by the NeRF for the $m$-th pixel.
- $x_m$: 3D point where the pixel lies.
- $\omega_m$: Corresponding viewing direction.
- $M$: Total number of training samples [S1].

**Stage 2: Ensemble Fine-Tuning (Ens-FT)**
Fine-tune the ensemble of NeRFs by optimizing the averaged output [S1]. The ensemble output $C_{avg}$ is the simple average of the outputs of the two NeRF models:

$$
C_{avg} = \frac{1}{2} \sum_{n=1}^{2} C^n(w_n, x, \omega)
$$

Where $C^n$ is the output of the $n$-th NeRF [S1]. The loss function minimized during this stage is the difference between the ground truth $C_m$ and the averaged ensemble output [S1]. This stage continues until performance drops [S1].

**Stage 3: Conjoint Pruning (CPE)**
Compress the ensemble using the validation set $D_{val}$ via conjoint pruning [S1]. This step is critical to prevent performance degradation that occurs with vanilla ensembling [S1]. The pruning is performed jointly on both models to maintain the ensemble's quality under parameter constraints [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_nerf_ensemble_with_conjoint_pruning(D_train, D_val):
    # Stage 1: Independent Training
    # Train two NeRFs with different grid resolutions independently
    model_1 = initialize_nerf(grid_resolution="high")
    model_2 = initialize_nerf(grid_resolution="low")
    
    # Minimize L(w) for both models on D_train
    model_1 = train_independently(model_1, D_train)
    model_2 = train_independently(model_2, D_train)
    
    ensemble = [model_1, model_2]
    
    # Stage 2: Ensemble Fine-Tuning (Ens-FT)
    # Fine-tune the ensemble by optimizing the averaged output
    while performance_has_not_dropped(ensemble, D_val):
        for model in ensemble:
            # Compute ensemble output C_avg = (C^1 + C^2) / 2
            # Minimize loss between C_m and C_avg
            model = fine_tune_ensemble_step(model, ensemble, D_train)
            
    # Stage 3: Conjoint Pruning (CPE)
    # Compress the ensemble using D_val
    ensemble = conjoint_pruning(ensemble, D_val)
    
    return ensemble
```

### Parameters
- **Ensemble Size**: 2 NeRF models [S1].
- **Grid Resolution**: The two models use different grid resolutions [S1].
- **Training Iterations**: Fixed number of iterations for Stage 1 [S1].

### Edge Cases and Pitfalls
- Vanilla ensembling of two models may yield suboptimal results if not combined with conjoint pruning and fine-tuning [S1].
- The size of the generated model may vary depending on the training process, requiring careful management of memory footprint [S1].

## Operating characteristics
- **Time and Memory Complexity**: The method requires training two models independently, which increases initial training time compared to a single model. However, it maintains a similar parameter count and memory footprint compared to a single large model [S1].
- **Performance**: Experimental results show that the ensemble approach beats vanilla approaches under the same parameter’s cardinality constraint and achieves superior results compared to state-of-the-art architectures [S1].
- **Scaling**: Ensembling multiple models at different scales requires fewer parameters than training and compressing one large model directly under the same generated image quality constraint [S1].
- **Strengths**: Improves novel view synthesis quality without increasing memory footprint significantly [S1].
- **Limitations**: Requires careful tuning of the conjoint pruning phase to avoid performance degradation [S1].

## Background
The method was proposed by Francesco Di Sario, Riccardo Renzulli, Enzo Tartaglione, and Marco Grangetto in 2023 for ICIAP 2023 [S1]. It builds on the Direct Voxel Grid Optimization (DVGO) architecture, which is a popular baseline for NeRFs due to its simplicity and effectiveness [S1]. The work is the first to propose a joint ensembling and compression scheme for NeRF models, addressing the performance degradation issues in vanilla ensembling [S1].

## Sources
[S1] Two is Better than One: Achieving High-Quality 3D Scene Modeling with a NeRF Ensemble, https://doi.org/10.1007/978-3-031-43153-1_27, fetched 2026-10-26
