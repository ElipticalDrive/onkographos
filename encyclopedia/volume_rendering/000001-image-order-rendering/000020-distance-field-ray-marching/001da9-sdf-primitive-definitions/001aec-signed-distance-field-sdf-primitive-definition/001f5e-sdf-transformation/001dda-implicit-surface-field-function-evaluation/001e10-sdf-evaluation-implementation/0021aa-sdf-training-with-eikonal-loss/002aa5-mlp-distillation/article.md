# MLP Distillation

Also called: Knowledge Distillation for SDF, SDF Distillation

## Overview
MLP Distillation compresses a large, high-capacity Signed Distance Function (SDF) network into a smaller, more efficient MLP while preserving geometric accuracy. It solves the problem of high inference latency and memory usage associated with large neural implicit representations by transferring knowledge from a "teacher" model to a "student" model. Unlike standard training which learns geometry from scratch, this method minimizes the output discrepancy between the two networks at sampled points, allowing the small network to inherit the teacher's geometric priors. It sets itself apart from sibling methods like Sphere Loss by focusing on direct output-space alignment rather than geometric constraints on the field's curvature or local shape.

## How it works
**Inputs and Outputs:**
- **Input:** A pre-trained large SDF network $f$ (teacher) with channel width 256, and a randomly initialized small SDF network $f_{\text{small}}$ (student) with channel width 16. A set of training rays $\mathbf{r}$ sampled from the training dataset.
- **Output:** The weights of the distilled small SDF network $f_{\text{small}}$.

**Data Structures:**
- **Teacher Network $f$:** Multi-Layer Perceptron (MLP) with 256 hidden units.
- **Student Network $f_{\text{small}}$:** Multi-Layer Perceptron (MLP) with 16 hidden units.
- **Sampled Points $\mathbf{x}_i$:** 3D coordinates along the training rays.

**Parameters:**
- **Distillation Iterations:** 5,000 iterations.
- **Loss Function:** L1 loss.
- **Gradient Handling:** Stop gradient applied to teacher outputs.

**Algorithm:**
1. **Sampling:** For each of the 5,000 distillation iterations, sample a batch of random rays $\mathbf{r}$ from the training set.
2. **Point Evaluation:** For each ray, evaluate the teacher network $f$ and the student network $f_{\text{small}}$ at the sampled points $\mathbf{x}_i$ along the ray.
3. **Loss Calculation:** Compute the distillation loss $\mathcal{L}_{\text{dist}}$ using the L1 norm of the difference between the teacher's and student's SDF outputs. A stop gradient operation is applied to the teacher's output to prevent it from updating during this phase.
   $$
   \mathcal{L}_{\text{dist}}(\mathbf{r}) := \sum_{i=0}^{N-1} |f(\mathbf{x}_i) - f_{\text{small}}(\mathbf{x}_i)|
   $$
   where $N$ is the number of sampled points per ray.
4. **Optimization:** Update the weights of $f_{\text{small}}$ by minimizing $\mathcal{L}_{\text{dist}}$ using gradient descent.
5. **Replacement:** After 5,000 iterations, discard the original teacher network $f$ and replace it with the distilled network $f_{\text{small}}$ for subsequent finetuning stages.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialization
teacher_model = LargeSDFMLP(channels=256) # Pre-trained
student_model = SmallSDFMLP(channels=16) # Random initialization
distill_iterations = 5000

# Distillation Loop
for iteration in range(distill_iterations):
    # 1. Sample random rays from training set
    rays = sample_training_rays(batch_size=B)
    
    # 2. Evaluate networks at sampled points
    # Assume rays contain points x_i
    points = rays.points
    
    # Teacher forward pass (no gradient needed for student update)
    with torch.no_grad():
        teacher_outputs = teacher_model(points)
        
    # Student forward pass
    student_outputs = student_model(points)
    
    # 3. Compute L1 Loss
    # Stop gradient is implicitly handled by torch.no_grad() on teacher
    loss = torch.mean(torch.abs(teacher_outputs - student_outputs))
    
    # 4. Optimize student model
    student_optimizer.zero_grad()
    loss.backward()
    student_optimizer.step()

# 5. Replace teacher with student for finetuning
final_sdf_model = student_model
```

**Edge Cases and Pitfalls:**
- **Precision:** Ensure the student network is initialized with appropriate weights to avoid vanishing gradients during the initial distillation steps.
- **Boundaries:** The method assumes the sampled points $\mathbf{x}_i$ are within the domain where the teacher model provides meaningful SDF values. Points far from the surface may introduce noise if the teacher's SDF is not well-behaved in empty space.
- **Degenerate Input:** If the training rays do not cover the scene uniformly, the distilled model may inherit biases from the teacher's sampling distribution.

## Subtopics

### [Progressive Volume Distillation](7388-progressive-volume-distillation.md)
Progressive Volume Distillation (PVD) is a systematic method for converting trained Neural Radiance Field (NeRF) models between heterogeneous architectures, including MLPs, sparse tensors, low-rank tensors, and hash tables. It solves the problem of architectural incompatibility by constructing a unified two-level view of volume representations and performing distillation progressively from shallower to deeper levels. Unlike methods that distill only between identical architectures, PVD enables any-to-any conversion while maintaining teacher performance, leveraging a block-wise training strategy to accelerate the process significantly.

### [Probabilistic Directed Distance Field (PDDF) Training](9296-probabilistic-directed-distance-field-pddf-training.md)
Probabilistic Directed Distance Field (PDDF) Training is a method for learning directed distance fields (DDFs) from mesh data by allowing the depth output to be a probability distribution rather than a scalar. This approach solves the problem of representing discontinuous depth and visibility fields with a continuous ($C^1$) neural network, which is essential for handling occlusions and surface boundaries without artifacts. It sets itself apart from standard SDF training by explicitly modeling view-dependent discontinuities through a mixture of Dirac deltas, enabling differentiable rendering via ray-casting that depends only on a single forward pass.

## Operating characteristics
- **Time Complexity:** The distillation process requires 5,000 iterations of forward and backward passes through both the teacher and student networks. The teacher's forward pass is gradient-free, reducing computational cost compared to joint training.
- **Memory Complexity:** The student network uses significantly less memory than the teacher (16 channels vs. 256 channels), making it suitable for real-time rendering applications where memory bandwidth is a bottleneck.
- **Measured Performance:** The method enables the use of a small 16-channel MLP for inference while maintaining the geometric accuracy of a 256-channel MLP.
- **Strengths:** Significant reduction in model size and inference latency; preserves high-quality geometry learned by the teacher.
- **Limitations:** Requires a pre-trained large teacher model; distillation quality depends on the coverage of sampled points.
- **When to Use:** When real-time rendering is required but the initial training used a large, slow network.
- **When Not to Use:** When the teacher model is not available or when the scene geometry is too complex for a small MLP to capture even with distillation.

## Background
MLP Distillation was introduced in the context of HybridNeRF, an efficient neural rendering method that combines volumetric and surface-based approaches. The method addresses the challenge of complex backgrounds and the high cost of separate NeRF background models. By distilling a large SDF into a smaller one, it enables real-time rendering without doubling inference costs. The approach builds on SDF Training with Eikonal Loss, using the large network to learn accurate scene geometry before compression.

## Sources
[S1] HybridNeRF: Efficient Neural Rendering via Adaptive Volumetric Surfaces, https://doi.org/10.1109/cvpr52733.2024.01858, fetched 2026-10-26
