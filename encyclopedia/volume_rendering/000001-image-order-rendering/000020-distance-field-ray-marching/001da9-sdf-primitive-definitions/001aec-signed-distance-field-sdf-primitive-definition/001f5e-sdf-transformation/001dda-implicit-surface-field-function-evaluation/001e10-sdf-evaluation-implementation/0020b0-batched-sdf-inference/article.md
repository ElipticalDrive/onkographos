# Batched SDF Inference


## Overview
Batched SDF Inference evaluates Signed Distance Function (SDF) values for a large set of samples in batches to manage memory constraints [S1, unofficial]. It processes arbitrary query points sequentially in fixed-size chunks rather than evaluating them all at once or on a fixed lattice [S1, unofficial].

## How it works
**Inputs:**
- `samples`: A 2D array or list of lists of shape $(N, 3)$ containing $N$ query points in world space, where column 3 is reserved for the output SDF values [S1, unofficial].
- `decoder`: A function or neural network that computes the SDF value for a given point or batch of points.
- `max_batch`: Integer maximum batch size for inference, default $32^3$ [S1, unofficial].

**Outputs:**
- SDF values populated in column 3 of the `samples` array [S1, unofficial].

**Data Structures:**
- `samples`: A contiguous 2D array where the first three columns hold input coordinates and column 3 holds the computed SDF values [S1, unofficial].

**Steps:**
1. Disable gradients for samples [S1, unofficial].
2. Get number of samples, `num_samples` [S1, unofficial].
3. Initialize `head = 0` [G].
4. Loop while `head < num_samples`:
    a. Extract sample subset from `head` to `min(head + max_batch, num_samples)` [S1, unofficial].
    b. Decode SDF for subset and store in column 3 of the `samples` array [S1, unofficial].
    c. Increment `head` by `max_batch` [S1, unofficial].
5. Return SDF values from column 3 [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def batched_sdf_inference(samples, decode_sdf, max_batch=32**3):
    num_samples = len(samples)
    head = 0
    # Disable gradients for samples [S1, unofficial]
    while head < num_samples:
        batch_end = min(head + max_batch, num_samples)
        # Extract sample subset from head to batch_end [S1, unofficial]
        # Decode SDF for subset and store in column 3 [S1, unofficial]
        for i in range(head, batch_end):
            samples[i][3] = decode_sdf(samples[i][0], samples[i][1], samples[i][2])
        # Increment head by max_batch [S1, unofficial]
        head += max_batch
    # Return SDF values from column 3 [S1, unofficial]
    return [row[3] for row in samples]
```

**Parameters:**
- `max_batch`: The maximum number of samples processed in a single forward pass. The default value is $32^3$ (32,768) [S1, unofficial].

**Edge Cases and Pitfalls:**
Not given in the sources.

## Subtopics

### [SDF Value Computation](6443-sdf-value-computation.md)
A compute shader kernel that calculates the Signed Distance Function (SDF) value for each voxel in the grid. It transforms grid coordinates to world space, evaluates the SDF function, and stores the result.

## Operating characteristics
Not given in the sources.

## Background
The method originates from the DIST-Renderer project [S1, unofficial]. It is applied in neural implicit surface reconstruction pipelines to query millions of points for training and rendering [S1, unofficial].

## Sources
[S1] B1ueber2y/DIST-Renderer, https://github.com/B1ueber2y/DIST-Renderer, fetched 2026-10-26 (unofficial)
