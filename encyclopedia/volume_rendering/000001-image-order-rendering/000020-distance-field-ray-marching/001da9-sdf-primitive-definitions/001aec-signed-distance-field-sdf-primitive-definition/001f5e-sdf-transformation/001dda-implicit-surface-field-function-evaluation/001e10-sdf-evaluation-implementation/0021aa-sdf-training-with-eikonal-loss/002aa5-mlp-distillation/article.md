# MLP Distillation

Also called: Knowledge Distillation for SDF, SDF Distillation

## Overview
MLP Distillation compresses a high-capacity Signed Distance Function (SDF) network into a smaller, more efficient network while preserving geometric accuracy. It solves the inference latency and memory overhead of large MLPs by transferring knowledge from a teacher model to a student model. Unlike [Sphere Loss](2750-sphere-loss.md), which enforces geometric constraints via local sphere approximations, distillation directly minimizes the output discrepancy between the two networks.

## How it works
**Inputs:**
- Teacher SDF network $f$ (large capacity).
- Student SDF network $f_{\text{small}}$ (small capacity).
- Training ray set $\mathcal{R}$.

**Outputs:**
- Distilled student network $f_{\text{small}}$.

**Data Structures:**
- Teacher MLP: 256 channels per hidden layer [S1].
- Student MLP: 16 channels per hidden layer [S1].
- Sampled points $\mathbf{x}_i$ along rays $\mathbf{r} \in \mathcal{R}$.

**Parameters:**
- Distillation iterations: 5,000 [S1].
- Loss function: L1 [S1].

**Steps:**
1. Sample random rays from the training set for 5,000 iterations [S1].
2. For each ray, evaluate the teacher network $f(\mathbf{x}_i)$ and student network $f_{\text{small}}(\mathbf{x}_i)$ at sampled points $\mathbf{x}_i$.
3. Compute the distillation loss $\mathcal{L}_{\text{dist}}$ using the L1 norm between the teacher and student outputs, applying a stop gradient to the teacher's output to prevent backpropagation into it [S1]:
   $$
   \mathcal{L}_{\text{dist}}(\mathbf{r}) := \sum_{i=0}^{N-1} |f(\mathbf{x}_i) - f_{\text{small}}(\mathbf{x}_i)|
   $$
4. Update the weights of $f_{\text{small}}$ to minimize $\mathcal{L}_{\text{dist}}$.
5. After distillation, discard the original teacher network $f$ and use $f_{\text{small}}$ for the subsequent finetuning stage [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Assume teacher_model and student_model are initialized
# teacher_model has 256 channels, student_model has 16 channels

distillation_iterations = 5000
optimizer = Adam(student_model.parameters())

for iteration in range(distillation_iterations):
    # Sample random rays from training set
    rays = sample_random_rays(training_set)
    
    total_loss = 0.0
    
    for ray in rays:
        # Sample points along the ray
        points = sample_points_along_ray(ray)
        
        # Forward pass through teacher (no gradient needed)
        with torch.no_grad():
            teacher_outputs = teacher_model(points)
        
        # Forward pass through student
        student_outputs = student_model(points)
        
        # Compute L1 loss
        loss = torch.mean(torch.abs(teacher_outputs - student_outputs))
        
        # Backward pass and update student
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
        
        total_loss += loss.item()
    
    # Optional: log total_loss
```

**Edge Cases and Pitfalls:**
- The teacher network must be fully trained before distillation begins [S1].
- The stop gradient is critical; without it, the teacher's weights would also update, potentially destabilizing the knowledge transfer [S1].
- Precision issues may arise if the student network is too small to capture the teacher's geometry; 16 channels is specified as sufficient in the source [S1].

## Subtopics

### [Progressive Volume Distillation](7388-progressive-volume-distillation.md)
A systematic distillation method that allows any-to-any conversions between different NeRF architectures (MLP, sparse tensors, low-rank tensors, hashtables) by performing distillation progressively on different levels of volume representations, from shallower to deeper.

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of sampled points and iterations. The distillation phase adds 5,000 iterations of training overhead [S1].
- **Memory Complexity:** The student network uses significantly less memory than the teacher (16 channels vs. 256 channels) [S1].
- **Performance:** Enables real-time rendering by reducing inference cost, as the distilled model replaces the large teacher network [S1].
- **Strengths:** Simple to implement, effective at preserving geometry, reduces model size.
- **Limitations:** Requires a pre-trained large teacher network; distillation quality depends on the capacity of the student network.

## Background
MLP Distillation is used in [HybridNeRF](https://doi.org/10.1109/cvpr52733.2024.01858) to compress the SDF representation learned during the initial training phase. The authors found that a large 256-channel MLP is necessary for accurate geometry learning, but a smaller 16-channel network suffices for inference after distillation [S1]. This approach avoids the need for separate background models, which double inference and memory costs [S1].

## Sources
[S1] HybridNeRF: Efficient Neural Rendering via Adaptive Volumetric Surfaces, https://doi.org/10.1109/cvpr52733.2024.01858, fetched 2026-10-26
