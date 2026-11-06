# SDF Validity Check with Relaxation


## Overview
This method filters Signed Distance Field (SDF) samples to distinguish between valid near-surface values and invalid or noisy estimates by applying a relaxation threshold relative to the voxel size. It solves the problem of ray marching through regions where the SDF magnitude is ambiguous, preventing artifacts caused by treating noisy near-zero values as exact surface intersections. It differs from [Batched SDF Inference](2329-batched-sdf-inference.md) by focusing on post-evaluation validation rather than generation, and from [Eikonal SDF Refinement](4597-eikonal-sdf-refinement.md) by using a simple threshold check rather than solving the Eikonal equation for gradient correction.

## How it works
**Inputs:**
- `sdf_values`: Array of SDF values for a batch of samples.
- `voxel_size`: The spatial size of the voxel grid cell.
- `relaxation`: A dimensionless constant factor defining the validity threshold.

**Outputs:**
- `valid_mask`: Boolean array indicating which samples are within the valid relaxation band.
- `positive_mask`: Boolean array for samples with SDF > threshold.
- `negative_mask`: Boolean array for samples with SDF < -threshold.

**Data Structures:**
- `sdf_values`: 1D or N-dimensional array of floats.
- `valid_mask`, `positive_mask`, `negative_mask`: Boolean arrays of the same shape as `sdf_values`.

**Parameters:**
- `relaxation`: Default value is 1.5 [S1, unofficial].

**Steps:**
1. Calculate the relaxation threshold: $T = \text{voxel\_size} \times \text{relaxation}$.
2. Identify positive samples: those where $\text{sdf} > T$.
3. Identify negative samples: those where $-\text{sdf} > T$ (equivalent to $\text{sdf} < -T$).
4. Identify valid samples: those where $|\text{sdf}| \le T$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def check_valid(sdf_values, voxel_size, relaxation=1.5):
    """
    Classify SDF samples based on relaxation threshold.
    
    Args:
        sdf_values: Array of SDF values.
        voxel_size: Size of one voxel in world units.
        relaxation: Threshold factor (default 1.5).
        
    Returns:
        valid_mask: Samples near zero (|sdf| <= threshold).
        positive_mask: Samples far positive (sdf > threshold).
        negative_mask: Samples far negative (sdf < -threshold).
    """
    # Calculate threshold
    threshold = voxel_size * relaxation
    
    # Identify positive samples: SDF value > voxel_size * relaxation
    positive_mask = sdf_values > threshold
    
    # Identify negative samples: -SDF value > voxel_size * relaxation
    # Equivalent to sdf_values < -threshold
    negative_mask = -sdf_values > threshold
    
    # Identify valid samples: absolute SDF value <= voxel_size * relaxation
    valid_mask = abs(sdf_values) <= threshold
    
    return valid_mask, positive_mask, negative_mask
```

**Edge Cases and Pitfalls:**
- If `voxel_size` is zero, the threshold becomes zero, potentially misclassifying all non-zero values as invalid. [G] Guard against division by zero or invalid grid configurations.
- Precision issues near the threshold boundary may cause flickering if the SDF value oscillates around $T$. [G] Consider using a small epsilon for floating-point comparisons if stability is required.

## Subtopics

### [Eikonal SDF Refinement](4597-eikonal-sdf-refinement.md)
Corrects inaccurate SDF values near the zero-level set by solving the Eikonal equation $|\nabla \phi| = 1$ over neighboring voxels. It sets itself apart by using an analytical solution to ensure the gradient magnitude is exactly 1, rather than just smoothing values. It builds on this method.

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of samples. The operation involves simple arithmetic comparisons.
- **Memory Complexity:** $O(N)$ for storing the three boolean masks.
- **Strengths:** Computationally inexpensive; effectively filters out noisy or ambiguous SDF values that could cause ray marching artifacts.
- **Limitations:** Does not correct the SDF values, only classifies them. May discard valid samples if the relaxation factor is too small or the voxel size is inaccurate.
- **When to use:** When post-processing SDF evaluations to ensure robust ray marching, especially in neural or implicit representations where noise is present.
- **When not to use:** When precise surface reconstruction is required without subsequent refinement steps.

## Background
This method is derived from the DIST-Renderer project, which implements distance-based rendering techniques. It is used in the context of synthesizing 3D data from 2D images, particularly in neural rendering pipelines where SDFs are evaluated during training or inference. The relaxation factor of 1.5 is a heuristic value chosen to balance between accepting valid near-surface samples and rejecting noise.

## Sources
[S1] B1ueber2y/DIST-Renderer, https://github.com/B1ueber2y/DIST-Renderer, fetched 2026-10-26 (unofficial)
