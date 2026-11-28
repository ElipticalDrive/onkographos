# Frozen Latent Code

Also called: Fixed Latent Code Optimization, Latent Code Freezing

## Overview
Frozen Latent Code is a finetuning variant in differentiable volume rendering where the per-identity latent code is kept fixed while only the model parameters are optimized [S1]. It solves the problem of adapting a pretrained volumetric prior to a target identity by reducing the optimization degrees of freedom compared to joint optimization [S1]. Unlike [Model Fitting (Fine-tuning)](18287-model-fitting-fine-tuning.md), which jointly optimizes the latent code and model weights, this method optimizes only the model parameters, yielding visually similar results but with different quantitative metrics [S1].

## How it works
**Inputs:**
- Pretrained volumetric prior model with weights $\theta_{\text{pretrained}}$ [S1].
- Target identity's latent code $w_{\text{target}}$, obtained from an initialization step (e.g., inversion) [S1].
- Sparse input views of the target identity [S1].

**Outputs:**
- Finetuned model parameters $\theta_{\text{target}}$ [S1].

**Data Structures:**
- Model parameters $\theta_{\text{target}}$: Tensor of weights for the volumetric prior network [S1].
- Latent code $w_{\text{target}}$: Fixed vector representing the target identity's features [S1].

**Steps:**
1. Initialize $\theta_{\text{target}} \leftarrow \theta_{\text{pretrained}}$ [S1].
2. Fix $w_{\text{target}}$ as a constant; do not add it to the optimizer's parameter list [S1].
3. For each training iteration:
   a. Render images from novel viewpoints using $\theta_{\text{target}}$ and fixed $w_{\text{target}}$ [S1].
   b. Compute loss against input views [S1].
   c. Backpropagate gradients to update $\theta_{\text{target}}$ only [S1].
4. Return $\theta_{\text{target}}$ [S1].

**Formulas:**
The optimization objective is:
$$ \arg \min_{\theta_{\text{target}}} \mathcal{L}(\theta_{\text{target}}, w_{\text{target}}) $$
where $w_{\text{target}}$ is constant [S1].

**Parameters:**
- Resolution: 1024 × 1024 for metric evaluation [S1].
- Subjects: 3 holdout subjects used for ablation [S1].

**Edge cases and pitfalls:**
- If geometry is not initialized correctly at the start, artifacts such as a second ear, unrealistic forehead, or fuzzy surface may occur [S1]. Starting from an inversion result mitigates these artifacts [S1].
- The method may fail on out-of-distribution facial expressions if the model was trained on neutral faces with closed mouths [S1].

## Subtopics

### [Model Fitting (Fine-tuning)](18287-model-fitting-fine-tuning.md)
Jointly optimizes latent code and model weights. Choose when higher quantitative metrics are needed despite similar visual quality.

## Operating characteristics
- **Time and memory complexity:** Not given in the sources.
- **Measured performance:** Quantitative metrics were computed for three holdout subjects at 1024 × 1024 resolution [S1]. Visual results are very similar to joint optimization [S1].
- **Strengths and limitations:** Reduces optimization complexity by fixing the latent code [S1]. May yield different quantitative metrics compared to joint optimization [S1].
- **When to use:** When visual fidelity is comparable to joint optimization but simpler optimization is preferred [S1].
- **When not:** When joint optimization provides significantly better quantitative metrics [S1].

## Background
Originated in "A Data-driven Volumetric Prior for Few-shot Ultra High-resolution Face Synthesis" [S1]. It is used for few-shot ultra high-resolution face synthesis from sparse input views [S1].

## Sources
[S1] Preface: A Data-driven Volumetric Prior for Few-shot Ultra High-resolution Face Synthesis, https://doi.org/10.1109/iccv51070.2023.00315, fetched 2026-10-27
