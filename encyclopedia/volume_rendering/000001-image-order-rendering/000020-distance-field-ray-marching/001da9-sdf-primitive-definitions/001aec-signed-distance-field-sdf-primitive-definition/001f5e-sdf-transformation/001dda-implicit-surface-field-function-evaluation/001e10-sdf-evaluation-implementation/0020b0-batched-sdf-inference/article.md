# Batched SDF Inference

Also called: Batched SDF Evaluation, SDF Batch Inference

## Overview
Batched SDF Inference evaluates Signed Distance Function (SDF) values for a large set of spatial samples by processing them in fixed-size chunks to manage GPU memory constraints [S1, unofficial]. It differs from single-sample evaluation by optimizing throughput for dense grids, and from training methods by being a purely inferential, non-gradient step [S1, unofficial].

## How it works
**Inputs:**
- `samples`: A tensor of shape $(N, D)$ containing $N$ query points in $D$ dimensions (typically $D=3$).
- `decoder`: A pre-trained implicit shape decoder (e.g., a neural network or analytic function) capable of evaluating the SDF for a specific shape instance.
- `max_batch`: Integer maximum batch size for inference. Default value is $32^3 = 32768$ [S1, unofficial].

**Outputs:**
- `sdf_values`: A tensor of shape $(N,)$ containing the computed SDF values for each sample.

**Data Structures:**
- The `samples` tensor is indexed by a `head` pointer.
- The output is stored in a pre-allocated tensor or appended to a list, specifically mapped to "column 3" in the source implementation, implying a structured buffer where SDF values are written to a specific index [S1, unofficial].

**Steps:**
1.  Disable gradient computation for the current inference pass to save memory and computation time [S1, unofficial].
2.  Determine the total number of samples, $N$ [S1, unofficial].
3.  Initialize a loop with `head = 0`.
4.  While `head < N`:
    a.  Calculate the end of the current batch: `end = min(head + max_batch, N)` [S1, unofficial].
    b.  Extract the subset of samples: `batch_samples = samples[head:end]` [S1, unofficial].
    c.  Decode the SDF for this subset using the shape decoder [S1, unofficial].
    d.  Store the resulting SDF values in the output buffer at the corresponding indices (specifically "column 3" as per the source implementation) [S1, unofficial].
    e.  Increment `head` by `max_batch` [S1, unofficial].
5.  Return the full array of SDF values [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def batched_sdf_inference(samples, decoder, max_batch=32**3):
    """
    Evaluates SDF for samples in batches.
    
    Args:
        samples: Tensor of shape (N, 3) containing query points.
        decoder: Function or model that returns SDF value given points and latent code.
        max_batch: Integer, maximum number of samples per inference step.
        
    Returns:
        Tensor of shape (N,) containing SDF values.
    """
    num_samples = samples.shape[0]
    sdf_values = torch.empty(num_samples) # Pre-allocate output buffer [G]
    
    # Disable gradients for inference [S1, unofficial]
    with torch.no_grad():
        head = 0
        while head < num_samples:
            # Determine batch end [S1, unofficial]
            end = min(head + max_batch, num_samples)
            
            # Extract subset [S1, unofficial]
            batch_points = samples[head:end]
            
            # Decode SDF for subset [S1, unofficial]
            # Note: The source implies a specific decoder structure where 
            # shape codes are loaded separately. Here we assume decoder 
            # accepts points and returns scalar distances.
            batch_sdf = decoder(batch_points)
            
            # Store in column 3 / output buffer [S1, unofficial]
            # Assuming batch_sdf is 1D or squeezed to 1D
            sdf_values[head:end] = batch_sdf
            
            # Increment head [S1, unofficial]
            head += max_batch
            
    return sdf_values
```

**Parameters:**
- `max_batch`: Default $32^3$ [S1, unofficial].

**Edge Cases:**
- If `num_samples` is 0, the loop does not execute, and an empty tensor is returned.
- Precision issues may arise if `max_batch` is too large for the available VRAM, causing OOM errors. The default $32^3$ is a heuristic balance [S1, unofficial].

## Subtopics

### [SDF Value Computation](6443-sdf-value-computation.md)
The core kernel that evaluates the SDF for a grid of voxels, transforming local shape coordinates to world space. It serves as the atomic evaluation step within batched inference pipelines, providing the geometric data required for subsequent ray marching or surface extraction. It differs from multi-view fusion methods by operating on a single implicit shape definition at a time, focusing purely on the mathematical evaluation of the distance field rather than photometric consistency or view-dependent projection.

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of samples, with a constant factor dependent on the decoder's inference cost per sample.
- **Memory Complexity:** $O(max\_batch)$ for the active inference buffer, plus $O(N)$ for the output storage. This is significantly lower than evaluating all $N$ samples simultaneously if $N \gg max\_batch$.
- **Performance:** The default batch size of $32^3$ is chosen to fit within typical GPU memory limits while maximizing throughput [S1, unofficial].
- **Strengths:** Prevents out-of-memory errors during dense grid evaluation; simple to implement.
- **Limitations:** Does not optimize for cache locality across batches; requires pre-allocation of output memory.

## Background
This method is derived from the DIST-Renderer implementation, which uses batched inference to evaluate latent-coded SDFs for 3D shape reconstruction [S1, unofficial]. It is part of a pipeline that includes camera list generation and latent code loading for shape and color decoding.

## Sources
[S1] B1ueber2y/DIST-Renderer, https://github.com/B1ueber2y/DIST-Renderer, fetched 2026-10-26 (unofficial)
