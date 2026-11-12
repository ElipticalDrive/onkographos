# Parallelization for Backpropagation
Also called: Per-splat Backpropagation

## Overview
Parallelization for Backpropagation is a strategy for computing gradients in 3D Gaussian Splatting that shifts the computational granularity from per-pixel to per-splat (per-Gaussian) [S1]. It addresses training speed obstacles by restructuring the backward pass to derive faster, numerically equivalent solutions for gradient computation and attribute updates [S1].

## How it works
The method replaces the standard pixel-parallel gradient accumulation with a per-splat iteration order [S1].

**Inputs:** Not given in the sources [S1].
**Outputs:** Not given in the sources [S1].
**Data Structures:** Not given in the sources [S1].

**Steps:**
1. The exact algorithmic steps for the per-splat backward pass are not provided in the sources [S1].
2. The method computes gradients and updates attributes using a numerically equivalent but faster formulation [S1].
3. # Pseudocode is not provided in the sources [S1].

**Formulas:** Not given in the sources [S1].
**Parameters:** Not given in the sources [S1].
**Edge Cases and Pitfalls:** Not given in the sources [S1].

## Subtopics

### [Per-Gaussian Backward Pass](9590-per-gaussian-backward-pass.md)
The Per-Gaussian Backward Pass is a parallelization strategy for the backward pass of 3D Gaussian Splatting that computes gradients by iterating over Gaussians rather than pixels. It solves the high atomic contention and false sharing inherent in standard pixel-parallel backpropagation. By reducing the number of required atomic operations by a factor equal to the number of pixels in a tile (typically 256 for 16×16 tiles), it significantly accelerates training time at the cost of increased VRAM usage.

## Operating characteristics
- **Time Complexity:** Reduces training time by 4–5× compared to standard approaches in budgeted settings [S1].
- **Memory Complexity:** Not given in the sources [S1].
- **Strengths:** Provides an alternative parallelization strategy for efficient backpropagation [S1].
- **Limitations:** Not given in the sources [S1].
- **When to Use:** Not given in the sources [S1].

## Background
The method was introduced in "Taming 3DGS: High-Quality Radiance Fields with Limited Resources" as part of a broader effort to optimize the 3D Gaussian Splatting training pipeline [S1]. It addresses scalability issues by rethinking the parallelization strategy of the backward pass [S1].

## Sources
[S1] Taming 3DGS: High-Quality Radiance Fields with Limited Resources, https://doi.org/10.1145/3680528.3687694, fetched 2026-10-30
