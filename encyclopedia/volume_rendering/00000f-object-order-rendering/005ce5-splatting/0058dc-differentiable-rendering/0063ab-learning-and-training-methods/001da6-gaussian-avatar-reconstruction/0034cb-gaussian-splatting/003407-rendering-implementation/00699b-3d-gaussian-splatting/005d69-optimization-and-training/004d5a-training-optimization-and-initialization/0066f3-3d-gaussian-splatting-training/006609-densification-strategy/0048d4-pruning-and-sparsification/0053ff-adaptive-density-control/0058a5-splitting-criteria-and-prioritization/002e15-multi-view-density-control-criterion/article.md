# Multi-View Density Control Criterion

Also called: Absolute-gradient split proxy, FastGS criterion, Multi-view Adaptive Density Control (ADC)

## Overview
The Multi-View Density Control Criterion is a densification strategy for 3D Gaussian Splatting designed to resolve invalid vector arithmetic that occurs when applying standard Adaptive Density Control (ADC) to multi-view training. Standard ADC sums 2D positional gradients from pixels covering a Gaussian; however, in multi-view training, a single Gaussian is projected onto multiple image planes, and adding 2D gradient vectors from different view spaces is mathematically invalid, potentially resulting in zero vectors even if the underlying 3D gradients are significant [S1, unofficial]. This method replaces vector addition with the summation of squared norms, ensuring that error metrics remain valid across viewpoints [S1, unofficial]. It yields higher reconstruction quality with fewer primitives compared to standard positional-gradient rules [S2, unofficial].

## How it works
The method computes a scalar error score $E_1(G)$ for each Gaussian $G$ by aggregating the squared magnitudes of gradients from all pixels that project onto $G$ across all viewpoints.

**Inputs:**
- $N$: Total number of viewpoints (images) used in the training batch.
- $G$: A specific 3D Gaussian primitive.
- $\mathcal{P}_k$: The set of pixels in viewpoint $k$ that are covered by the projection of $G$.
- $\nabla_{\pi} L$: The 2D positional gradient of the loss $L$ with respect to the pixel coordinates $(u, v)$ for pixel $\pi$.

**Data Structures:**
- No specific complex data structures are required beyond standard gradient buffers. The criterion operates on the scalar values derived from the gradient tensors.

**Algorithm:**
1. Initialize the error score for Gaussian $G$ to zero: $E_1(G) \leftarrow 0$.
2. Iterate over each viewpoint $k$ from $1$ to $N$.
3. For each pixel $\pi$ in the set $\mathcal{P}_k$ (pixels covered by $G$ in viewpoint $k$):
   a. Retrieve the 2D positional gradient $\nabla_{\pi} L$.
   b. Compute the squared Euclidean norm of this gradient: $\|\nabla_{\pi} L\|^2$.
   c. Add this squared norm to the error score: $E_1(G) \leftarrow E_1(G) + \|\nabla_{\pi} L\|^2$.
4. Compare $E_1(G)$ against a threshold $\tau$ (typically derived from the parent splitting criteria logic). If $E_1(G) > \tau$, mark $G$ for densification (splitting or cloning).

**Formula:**
$$
E_1(G) = \sum_{k=1}^{N} \sum_{\pi \in \mathcal{P}_k} \|\nabla_{\pi} L\|^2
$$
Where:
- $N$ is the number of viewpoints.
- $\mathcal{P}_k$ is the set of pixels in viewpoint $k$ covered by $G$.
- $\nabla_{\pi} L$ is the 2D positional gradient at pixel $\pi$.
- $\|\cdot\|$ denotes the Euclidean norm.

**Parameters:**
- The threshold $\tau$ is not defined by this specific criterion in isolation but is passed from the parent [Splitting Criteria and Prioritization](16226-splitting-criteria-and-prioritization.md). The specific hyperparameter controlling the constraint strength in related implementations is the DS Threshold (DSTH), with a default value of 30 [S3, unofficial].

**Edge Cases and Pitfalls:**
- **Invalid Vector Addition:** The primary pitfall avoided by this method is the direct summation of 2D gradient vectors $\sum \nabla_{\pi} L$ from different viewpoints. As shown in [S1, unofficial], adding 2D gradients from different image planes (different coordinate spaces) can result in cancellation (e.g., two identical 3D gradients producing 2D gradients that sum to zero), leading to incorrect densification decisions.
- **Precision:** Squared norms are always non-negative, preventing cancellation errors. However, large gradients may cause overflow if not handled with appropriate data types (e.g., float32 vs float64).
- **Boundary Conditions:** Pixels on the boundary of the Gaussian's support must be correctly identified to ensure $\mathcal{P}_k$ is accurate.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_multiview_error(G, viewpoints, gradients):
    """
    Computes the Multi-View Density Control error score for a Gaussian.
    
    Args:
        G: The Gaussian primitive object.
        viewpoints: List of viewpoint indices [0, ..., N-1].
        gradients: A dictionary or array where gradients[k][pi] is the 
                   2D positional gradient vector for pixel pi in viewpoint k.
                   
    Returns:
        E1: Scalar error score.
    """
    E1 = 0.0
    
    for k in viewpoints:
        # Get pixels covered by G in viewpoint k
        # Implementation detail: This requires the projection logic from the 
        # rendering pipeline to determine which pixels are covered by G.
        covered_pixels = get_covered_pixels(G, k)
        
        for pi in covered_pixels:
            # Retrieve 2D gradient vector for pixel pi in viewpoint k
            grad_vec = gradients[k][pi]
            
            # Compute squared Euclidean norm
            # grad_vec is typically a 2D vector [du, dv]
            squared_norm = grad_vec[0]**2 + grad_vec[1]**2
            
            # Accumulate
            E1 += squared_norm
            
    return E1
```

## Subtopics

### DS Threshold (DSTH)
A hyperparameter threshold that controls the densification constraints of Gaussians in the context of the Multi-View Density Control Criterion. Increasing DSTH imposes stricter constraints, reducing the number of Gaussians (NGS), rendering quality, and training time [S3, unofficial]. The default value is 30. For the playroom scene in the DeepBlending dataset, it is set to 60, while other scenes use 30 [S3, unofficial].

## Operating characteristics
- **Time Complexity:** The computation involves iterating over all pixels covering each Gaussian across all viewpoints. The cost is proportional to the number of covered pixels and viewpoints. It is generally faster than methods requiring complex vector operations or covariance updates during the scoring phase.
- **Memory Complexity:** Minimal additional memory is required as it operates on existing gradient buffers.
- **Performance:** The multi-view criterion yields an average improvement of 0.67 dB in PSNR and 0.028 in LPIPS compared to standard methods, while using 28% fewer primitives at the same rendering time [S2, unofficial].
- **Comparison:** In the "garden" scene, the standard positional-gradient rule produces 240,075 primitives with a PSNR of 26.83 and LPIPS of 0.1303. The multi-view criterion produces 148,658 primitives with a PSNR of 27.06 and LPIPS of 0.1247 [S2, unofficial].
- **Strengths:** Resolves the invalid arithmetic issue in multi-view training; reduces primitive count without sacrificing quality.
- **Limitations:** Requires access to per-pixel gradients for all viewpoints in the batch.
- **Usage:** Applied during the teacher training phase to control primitive count and distribution [S2, unofficial].

## Background
The method was proposed to address a fundamental incompatibility between standard 3D Gaussian Splatting's densification strategy and multi-view training pipelines. Standard Adaptive Density Control (ADC) relies on summing 2D positional gradients, which is valid for single-view or independent-view updates but fails when gradients from different view spaces are summed directly [S1, unofficial]. The Multi-View Density Control Criterion was introduced in "Efficient multi-view training for 3D Gaussian Splatting" [S1, unofficial] as a space-independent error metric. It is also referenced in the context of ultrafast spatial texture-atlas splatting [S2, unofficial] and loss-guided density control [S3, unofficial].

## Sources
[S1] Efficient multi-view training for 3D Gaussian Splatting, https://doi.org/10.48550/arxiv.2506.12727, fetched 2026-10-26 (unofficial)
[S2] Bake It Till You Make It: Ultrafast Spatial Texture-Atlas Splatting, https://doi.org/10.48550/arXiv.2607.13808, fetched 2026-10-29 (unofficial)
[S3] Loss-Guided Density Control and Efficient Backward Optimisation for Compact Gaussian Splatting, https://doi.org/10.21203/rs.3.rs-10572680/v1, fetched 2026-10-26 (unofficial)
