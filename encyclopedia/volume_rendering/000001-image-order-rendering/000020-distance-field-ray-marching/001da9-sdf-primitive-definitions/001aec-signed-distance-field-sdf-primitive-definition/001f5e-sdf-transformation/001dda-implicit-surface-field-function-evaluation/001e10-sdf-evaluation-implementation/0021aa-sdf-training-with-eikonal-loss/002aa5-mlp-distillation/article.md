# MLP Distillation

Also called: Knowledge Distillation for SDF, SDF Distillation

## Overview
MLP Distillation compresses a large Signed Distance Function (SDF) network into a smaller MLP by minimizing the output discrepancy between the two networks at sampled points [S1]. The teacher network $f$ uses 256 hidden channels, while the student network $f_{\text{small}}$ uses 16 hidden channels [S1]. The method transfers geometric knowledge from the teacher to the student over a fixed number of iterations before replacing the teacher for subsequent finetuning [S1].

## How it works
**Inputs and Outputs:**
- **Input:** A pre-trained large SDF network $f$ (teacher) with 256 hidden channels, a randomly initialized small SDF network $f_{\text{small}}$ (student) with 16 hidden channels, and a set of training rays $\mathbf{r}$ sampled from the training dataset [S1].
- **Output:** The updated weights of the distilled small SDF network $f_{\text{small}}$ [S1].

**Data Structures:**
- **Teacher Network $f$:** Multi-Layer Perceptron (MLP) with 256 hidden units [S1].
- **Student Network $f_{\text{small}}$:** Multi-Layer Perceptron (MLP) with 16 hidden units [S1].
- **Sampled Points $\mathbf{x}_i$:** 3D coordinates along the training rays [S1].

**Parameters:**
- **Distillation Iterations:** 5,000 iterations [S1].
- **Loss Function:** L1 loss [S1].
- **Gradient Handling:** Stop gradient applied to teacher outputs [S1].

**Algorithm:**
1. **Sampling:** For each of the 5,000 distillation iterations, sample a batch of random rays $\mathbf{r}$ from the training set [S1].
2. **Point Evaluation:** For each ray, evaluate the teacher network $f$ and the student network $f_{\text{small}}$ at the sampled points $\mathbf{x}_i$ along the ray [S1].
3. **Loss Calculation:** Compute the distillation loss $\mathcal{L}_{\text{dist}}$ using the L1 norm of the difference between the teacher's and student's SDF outputs [S1]. A stop gradient operation is applied to the teacher's output to prevent it from updating during this phase [S1].
   $$
   \mathcal{L}_{\text{dist}}(\mathbf{r}) := \sum_{i=0}^{N-1} |f(\mathbf{x}_i) - f_{\text{small}}(\mathbf{x}_i)|
   $$
   where $N$ is the number of sampled points per ray [S1].
4. **Optimization:** Update the weights of $f_{\text{small}}$ by minimizing $\mathcal{L}_{\text{dist}}$ using gradient descent [S1].
5. **Replacement:** After 5,000 iterations, discard the original teacher network $f$ and replace it with the distilled network $f_{\text{small}}$ for subsequent finetuning stages [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mlp_distillation(teacher_model, student_model, training_rays, iterations=5000):
    # teacher_model: MLP with 256 channels [S1]
    # student_model: MLP with 16 channels [S1]
    for iteration in range(iterations):
        # 1. Sample random rays from the training set [S1]
        rays_batch = sample_training_rays(training_rays)
        
        # 2. Extract 3D coordinates x_i along the rays
        points = rays_batch.points
        
        # 3. Evaluate teacher network with stop gradient [S1]
        teacher_outputs = teacher_model.forward(points, stop_gradient=True)
        
        # 4. Evaluate student network
        student_outputs = student_model.forward(points)
        
        # 5. Compute L1 loss: L_dist(r) = sum_{i=0}^{N-1} |f(x_i) - f_small(x_i)| [S1]
        loss = sum(abs(teacher_outputs - student_outputs))
        
        # 6. Update student weights via gradient descent
        gradients = compute_gradients(loss, student_model.parameters())
        student_model.update_weights(gradients)
        
    # 7. Discard original teacher and switch to distilled counterpart for finetuning [S1]
    return student_model
```

**Edge Cases and Pitfalls:**
- **Precision:** Ensure the student network is initialized with appropriate weights to avoid vanishing gradients during the initial distillation steps [G].
- **Boundaries:** The method assumes the sampled points $\mathbf{x}_i$ are within the domain where the teacher model provides meaningful SDF values [G].
- **Degenerate Input:** If the training rays do not cover the scene uniformly, the distilled model may inherit biases from the teacher's sampling distribution [G].

## Subtopics

### [Progressive Volume Distillation](7388-progressive-volume-distillation.md)
Progressive Volume Distillation (PVD) is a systematic method for converting trained Neural Radiance Field (NeRF) models between heterogeneous architectures, including MLPs, sparse tensors, low-rank tensors, and hash tables. It solves the problem of architectural incompatibility by constructing a unified two-level view of volume representations and performing distillation progressively from shallower to deeper levels. Unlike methods that distill only between identical architectures, PVD enables any-to-any conversion while maintaining teacher performance, leveraging a block-wise training strategy to accelerate the process significantly.

### [Probabilistic Directed Distance Field (PDDF) Training](9296-probabilistic-directed-distance-field-pddf-training.md)
Probabilistic Directed Distance Field (PDDF) Training is a method for learning implicit surface representations from oriented point clouds by supervising a neural network to output a probability distribution over depths rather than a single scalar value [G]. It solves the problem of representing discontinuous directed distance fields (DDFs) with a continuous ($C^1$) network, enabling accurate modeling of surface occlusions and visibility transitions [G]. It distinguishes itself from standard SDF training by explicitly modeling depth as a discrete mixture of delta functions and enforcing view consistency inequalities to ensure geometric coherence across different viewing directions [G].

## Operating characteristics
- **Time Complexity:** The distillation process requires 5,000 iterations of forward and backward passes through both networks [S1].
- **Memory Complexity:** The student network uses significantly less memory than the teacher (16 channels vs. 256 channels) [S1].
- **Measured Performance:** The method enables the use of a small 16-channel MLP for inference while maintaining the geometric accuracy of a 256-channel MLP [S1].
- **Strengths:** Significant reduction in model size and inference latency; preserves high-quality geometry learned by the teacher [G].
- **Limitations:** Requires a pre-trained large teacher model; distillation quality depends on the coverage of sampled points [G].
- **When to Use:** When real-time rendering is required but the initial training used a large, slow network [G].
- **When Not to Use:** When the teacher model is not available or when the scene geometry is too complex for a small MLP to capture even with distillation [G].

## Background
MLP Distillation was introduced in the context of HybridNeRF, an efficient neural rendering method that combines volumetric and surface-based approaches [S1]. The method addresses the challenge of complex backgrounds and the high cost of separate NeRF background models [S1]. By distilling a large SDF into a smaller one, it enables real-time rendering without doubling inference costs [S1]. The approach builds on SDF Training with Eikonal Loss, using the large network to learn accurate scene geometry before compression [S1].

## Sources
[S1] HybridNeRF: Efficient Neural Rendering via Adaptive Volumetric Surfaces, https://doi.org/10.1109/cvpr52733.2024.01858, fetched 2026-10-26
