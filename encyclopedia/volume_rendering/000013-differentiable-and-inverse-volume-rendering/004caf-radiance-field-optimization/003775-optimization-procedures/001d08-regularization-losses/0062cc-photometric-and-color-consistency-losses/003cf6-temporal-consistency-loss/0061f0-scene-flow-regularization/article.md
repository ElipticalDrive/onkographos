# Scene Flow Regularization

Also called: Scene Flow MLP, Scene flow regularization loss

## Overview
Scene Flow Regularization is a technique used in dynamic radiance field optimization to model 3D temporal motion by learning mappings between temporally neighboring time steps [S1, unofficial][S2]. It addresses inconsistent geometry and appearance in dynamic novel view synthesis by explicitly predicting 3D displacement vectors for every point and regularizing them directly [S1, unofficial][S2]. Unlike methods relying solely on 2D optical flow or implicit deformation fields, it provides explicit 3D flow supervision and enforces smoothness constraints on the predicted motion [S1, unofficial][S2].

## How it works
**Inputs and Outputs**
*   **Input:** 3D coordinates $(x, y, z)$ and a timestamp $t_i$ [S2].
*   **Output:** Two 3D scene flow vectors: $S_{i \rightarrow i+1}$ (displacement to the next frame) and $S_{i \rightarrow i-1}$ (displacement to the previous frame) [S2].

**Data Structures**
*   **Scene Flow MLP:** A multi-layer perceptron $\text{MLP}_{\theta_{sf}}$ that takes $(x, y, z, t_i)$ as input and outputs the concatenated flow vectors [S2]. The scene flow MLP is not part of the rendering process but is part of the loss computation [S2].

**Numbered Steps**
1.  **Query Scene Flow:** For a 3D point $\mathbf{p} = (x, y, z)$ at time $t_i$, compute the forward and backward scene flows using the scene flow MLP [S2]:
    $$ (S_{i \rightarrow i+1}, S_{i \rightarrow i-1}) = \text{MLP}_{\theta_{sf}}(x, y, z, t_i) $$
    where $S_{i \rightarrow i+1}$ is the 3D scene flow of point $(x,y,z)$ at time $t_i$ to the next frame, and $S_{i \rightarrow i-1}$ is to the previous frame [S2].
2.  **Compute Regularization Loss:** Calculate the smooth and small scene flow loss $\mathcal{L}_{\text{sf}}^{\text{reg}}$ to encourage spatial/temporal smoothness and small displacements [S1, unofficial][S2]:
    $$ \mathcal{L}_{\text{sf}}^{\text{reg}} = \| S_{i \rightarrow i+1} + S_{i \rightarrow i-1} \|_1 + \| S_{i \rightarrow i+1} \|_1 + \| S_{i \rightarrow i-1} \|_1 $$
3.  **Gradient Detachment:** Detach gradients from the dynamic radiance fields to the camera poses to prevent instability during optimization [S2].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def l1_norm(v):
    return sum(abs(v))

def compute_scene_flow_regularization_loss(mlp_sf, points, timestamps):
    # Step 1: Query the MLP for forward and backward scene flows
    # Output shape: (N, 6), where first 3 are S_{i->i+1} and last 3 are S_{i->i-1}
    flows = mlp_sf(points, timestamps)
    
    # Split into forward and backward flows
    S_forward = flows[:, :3]  # S_{i->i+1}
    S_backward = flows[:, 3:] # S_{i->i-1}
    
    # Step 2: Compute the regularization loss
    # Term 1: Forward-backward sum magnitude
    cycle_sum = S_forward + S_backward
    term1 = l1_norm(cycle_sum)
    
    # Term 2 & 3: Forward and backward magnitudes
    term2 = l1_norm(S_forward)
    term3 = l1_norm(S_backward)
    
    # Sum all terms
    loss = term1 + term2 + term3
    
    return loss
```

**Parameters**
*   **MLP Architecture:** Not given in the sources.
*   **Loss Weights:** Not given in the sources.

**Edge Cases and Pitfalls**
*   **Gradient Flow:** The scene flow MLP is separate from the rendering network. Gradients must be detached from the dynamic radiance fields to the camera poses to avoid optimization instability [S2].
*   **Temporal Neighborhood:** This method is restricted to small temporal neighborhoods; artifact-free novel-view synthesis is predominantly demonstrated on spatio-temporal camera trajectories close to the input trajectories [S1, unofficial].
*   **Integration for Large Displacements:** Some methods (e.g., NeRFlow) model infinitesimal displacements that require integration via Neural ODEs to obtain offsets, which differs from the direct flow prediction described here [S1, unofficial].

## Subtopics

### Spatio-Temporal Consistency Losses
Extends temporal consistency to include spatial smoothness constraints on the flow field. Choose when spatial coherence of motion is critical. See [Spatio-Temporal Consistency Losses](21620-spatio-temporal-consistency-losses.md).

## Operating characteristics
*   **Time Complexity:** Depends on the evaluation cost of the scene flow MLP per point [S2].
*   **Memory Complexity:** Requires storing the parameters of the scene flow MLP [S2].
*   **Strengths:** Encourages physically plausible motion, improves density prediction reasonableness, and supports forward-backward cycle consistency [S1, unofficial][S2].
*   **Limitations:** Accuracy depends on the quality of auxiliary data if used with optical flow or depth supervision (e.g., monocular depth estimation errors propagate) [S1, unofficial]. Restricted to small temporal neighborhoods for best results [S1, unofficial].
*   **When to use:** When optimizing dynamic radiance fields and 3D motion needs explicit regularization to prevent artifacts [S1, unofficial][S2].
*   **When not to use:** For scenes with large, non-local temporal displacements where infinitesimal flow assumptions fail or integration is required [S1, unofficial].

## Background
Originated in the context of dynamic neural radiance fields, with methods like [LNSW21], [XHKK21], [GSKH21], and [DZY*21] adopting scene-flow mappings to encourage consistency of reflectance and opacity across time [S1, unofficial]. The specific formulation using an MLP to predict forward/backward flows with cycle-consistency regularization is detailed in [S2].

## Sources
[S1] Advances in Neural Rendering, https://doi.org/10.48550/arxiv.2111.05849, fetched 2026-10-26 (unofficial)
[S2] Robust Dynamic Radiance Fields, https://doi.org/10.1109/cvpr52729.2023.00010, fetched 2026-10-26
