# SDF Loss

Also called: L1 SDF Loss, Signed Distance Field Loss

## Overview
SDF Loss is a regression loss function used to supervise neural networks predicting Signed Distance Fields (SDFs) by minimizing the L1 distance between predicted signed distances and ground truth SDF values [S1]. It is distinguished from occupancy-based losses (e.g., [Occupancy Networks](14234-occupancy-networks.md)) by directly optimizing geometric accuracy rather than binary classification, though it is often combined with sign supervision for stability in meta-learning contexts [S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Predicted signed distance $s$ and ground truth signed distance $s_{GT}$ at sampled points.
- Output: Scalar loss value $\mathcal{L}_s$.

**Data Structures:**
- $s$: Scalar tensor of predicted distances.
- $s_{GT}$: Scalar tensor of ground truth distances.

**Formula:**
The SDF Loss is defined as the L1 norm of the difference between the predicted and ground truth signed distances [S1]:

$$
\mathcal{L}_s(s) = \|s - s_{GT}\|_1
$$

**Parameters:**
- None inherent to the loss definition. Weights may be applied if combined with other losses (see Subtopics).

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sdf_loss(predicted_sdf: Tensor, ground_truth_sdf: Tensor) -> Tensor:
    """
    Computes the L1 loss between predicted and ground truth SDF values.
    
    Args:
        predicted_sdf: Tensor of predicted signed distances.
        ground_truth_sdf: Tensor of ground truth signed distances.
        
    Returns:
        Scalar tensor representing the mean L1 loss.
    """
    # Calculate absolute difference
    diff = abs(predicted_sdf - ground_truth_sdf)
    
    # Return mean over all elements
    return diff.mean()
```

**Edge Cases and Pitfalls:**
- **Clamping Instability:** Clamping L1 loss values during gradient-based meta-learning can lead to unstable training [S2, unofficial].
- **Precision:** Ensure ground truth SDFs are computed accurately (e.g., via ray marching against meshes) to avoid supervising with noisy targets [S1].

## Subtopics

### Composite Loss for MetaSDF
A multitask loss function designed for stable training of MetaSDF in 3D [S2, unofficial]. It addresses instability caused by clamping L1 loss in inner-loop meta-learning by tracking both SDF values and occupancy (the sign of the SDF) [S2, unofficial]. The network predicts two values: the distance to the surface and the sign of the distance function (inside or outside) [S2, unofficial]. The loss combines the L1 SDF loss with a binary cross-entropy sign loss, weighted using the uncertainty loss weighting scheme described by Kendall et al. [43] [S2, unofficial]. This strategy effectively combines the loss terms of DeepSDF (L1 SDF) and Occupancy Networks (sign/occupancy) [S2, unofficial]. It is critical for stable training in MetaSDF, whereas DeepSDF profits little from this combination [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of sampled points, as it involves a single pass of absolute difference and mean calculation.
- **Memory Complexity:** $O(N)$ to store the difference tensor before reduction.
- **Strengths:** Directly optimizes geometric accuracy; simple to implement; robust to outliers compared to L2 loss [S1].
- **Limitations:** Can be unstable in meta-learning if not combined with sign supervision or uncertainty weighting [S2, unofficial].
- **When to use:** When precise geometric reconstruction is required and ground truth SDFs are available [S1].
- **When not to use:** When training is unstable due to meta-learning inner loops without sign supervision [S2, unofficial].

## Background
The SDF Loss was introduced in the context of inverse rendering and SDF modeling, specifically in the FIRe (Fast Inverse Rendering Using Directional and Signed Distance Functions) framework [S1]. It is based on the principle of supervising neural networks to predict signed distances close to ground truth SDFs [S1]. The composite variant was developed for MetaSDF to enable stable meta-learning of SDFs by incorporating occupancy information [S2, unofficial].

## Sources
[S1] FIRe: Fast Inverse Rendering Using Directional and Signed Distance Functions, https://www.youtube.com/watch?v=Qj0ymDlRYZs, fetched 2026-10-30
[S2] MetaSDF: Meta-learning Signed Distance Functions, https://doi.org/10.48550/arxiv.2006.09662, fetched 2026-10-26 (unofficial)
