# Batched SDF Inference

Also called: Batched SDF Evaluation

## Overview
Batched SDF Inference evaluates Signed Distance Function values for a large set of spatial samples in batches to manage memory constraints during inference [S1, unofficial]. It differs from single-sample evaluation by processing multiple points simultaneously, and from training methods by focusing solely on forward evaluation without gradient accumulation [S1, unofficial].

## How it works
**Inputs:** A list of 3D query points $\mathbf{p}_i$, an SDF evaluation function, and `max_batch` (default $32^3$) [S1, unofficial].

**Outputs:** An array of SDF values $d(\mathbf{p}_i)$ corresponding to each input point [G].

**Data Structures:** `samples` is a 2D array of shape $(N, 4)$ where columns 0–2 store coordinates $(x, y, z)$ and column 3 stores the computed SDF value [G].

**Steps:**
1. Disable automatic differentiation for the inference process [S1, unofficial].
2. Determine the total number of samples $N$ [S1, unofficial].
3. Initialize a head pointer $h = 0$ [G].
4. While $h < N$:
   a. Calculate the batch end: $e = \min(h + \text{max\_batch}, N)$ [S1, unofficial].
   b. Extract the subset of samples from index $h$ to $e-1$ [S1, unofficial].
   c. Evaluate the SDF for this subset and store the results in column 3 of the `samples` array for indices $h$ to $e-1$ [S1, unofficial].
   d. Increment $h$ by $\text{max\_batch}$ [S1, unofficial].
5. Return the SDF values from column 3 of the `samples` array [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def batched_sdf_inference(samples, evaluate_sdf, max_batch=32**3):
    """
    Evaluates SDF values for a large set of samples in batches.
    
    Args:
        samples: Array of shape (N, 4) where columns 0-2 are coordinates and column 3 is for output.
        evaluate_sdf: Function that takes an array of points and returns an array of distances.
        max_batch: Integer maximum batch size. Default 32768.
        
    Returns:
        Array of SDF values of shape (N,).
    """
    # Disable gradients for memory efficiency [G]
    num_samples = len(samples)
    head = 0
    while head < num_samples:
        end = min(head + max_batch, num_samples)
        batch_points = [samples[i][:3] for i in range(head, end)]
        batch_sdfs = evaluate_sdf(batch_points)
        for i, val in enumerate(batch_sdfs):
            samples[head + i][3] = val
        head += max_batch
    return [sample[3] for sample in samples]
```

**Parameters:**
- `max_batch`: Integer maximum batch size for inference, default $32^3$ [S1, unofficial].

**Edge Cases and Pitfalls:**
- If `max_batch` exceeds $N$, the loop executes exactly once [G].
- Precision and memory limits depend entirely on the underlying SDF evaluation function and hardware constraints [G].

## Subtopics

### [SDF Value Computation](6443-sdf-value-computation.md)
A compute shader kernel that calculates the SDF value for each voxel in a grid [G]. It serves as the atomic evaluation step within batched inference pipelines [G].

### [Batched SDF Query with Joint Configuration](13037-batched-sdf-query-with-joint-configuration.md)
Evaluates SDFs for volumetric structures across multiple joint configurations and query points simultaneously, enabling efficient batched inference for robotic kinematics [G].

### [SDF Value and Gradient Query](13051-sdf-value-and-gradient-query.md)
Provides real-time inspection of SDF properties at specific 3D coordinates, outputting the signed distance value, analytical gradient, and hit classification [G].

## Operating characteristics
Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] B1ueber2y/DIST-Renderer, https://github.com/B1ueber2y/DIST-Renderer, fetched 2026-10-26 (unofficial)
