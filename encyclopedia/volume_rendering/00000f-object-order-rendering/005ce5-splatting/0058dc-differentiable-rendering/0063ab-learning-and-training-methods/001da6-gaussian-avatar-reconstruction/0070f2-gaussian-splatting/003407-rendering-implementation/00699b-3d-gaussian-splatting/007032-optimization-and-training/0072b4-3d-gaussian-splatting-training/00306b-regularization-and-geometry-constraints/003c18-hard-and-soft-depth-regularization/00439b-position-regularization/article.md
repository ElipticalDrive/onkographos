# Position Regularization
Also called: Position Constraint Loss, Surface Regularization Loss

## Overview
Position Regularization prevents floating artifacts and dramatic geometry shifts in 3D Gaussian Splatting by constraining Gaussian centers to remain close to a reference surface or their initial positions [S1][S2, unofficial]. It operates as a geometric prior constraint, distinct from depth-consistency methods in its parent category [9178-hard-and-soft-depth-regularization.md] [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Current Gaussian positions ($\mu$ or $p_i$), reference mesh vertices ($v_i$) or initial Gaussian positions ($\mu_{init}$), and hyperparameters ($\tau_{pos}$, $\lambda_{sr}$) [S1][S2, unofficial].
*   **Outputs:** A scalar loss value for gradient-based optimization [S1][S2, unofficial].

**Data Structures:**
*   Gaussian positions: Vector of 3D coordinates [S1][S2, unofficial].
*   Reference mesh vertices: Array of 3D coordinates from the SMPL or extracted mesh [S1][S2, unofficial].
*   Initial positions: Stored copy of Gaussian positions at initialization [S2, unofficial].

**Formulas:**
Two primary formulations exist in the literature [S1][S2, unofficial]:

1.  **Mesh-Proximity Loss (Human Gaussian Splatting):**
    $$L_{mesh} = \sum_i \text{ReLU}(\|v_i - p_i\|_2^2 - \tau_{pos})$$
    Where:
    *   $p_i$: Canonical position of the $i$-th Gaussian [S1].
    *   $v_i$: Nearest vertex on the reference mesh to $p_i$ [S1].
    *   $\tau_{pos}$: Threshold representing the maximum expected distance between skin and clothes [S1].
    *   $\text{ReLU}(x) = \max(0, x)$ [G].

2.  **Surface Regularization Loss (Phys3DGS):**
    $$L_{sr} = \|\mu - \mu_{init}\|_2^2$$
    Where:
    *   $\mu$: Current position of a Gaussian bound to the extracted mesh [S2, unofficial].
    *   $\mu_{init}$: Initial position of that Gaussian, stored separately at the start of training [S2, unofficial].

**Parameters:**
*   $\tau_{pos}$: Threshold for max distance between skin and clothes [S1].
*   $\lambda_{sr}$: Weight for the surface regularization loss in the total loss function [S2, unofficial].

**Steps:**
1.  Select the reference geometry: either the mesh vertices $v_i$ [S1] or the initial positions $\mu_{init}$ [S2, unofficial].
2.  For each Gaussian, find the nearest reference point $v_i$ or retrieve its corresponding $\mu_{init}$ [S1][S2, unofficial].
3.  Compute the squared Euclidean distance between the current Gaussian position and the reference point [S1][S2, unofficial].
4.  Apply the specific loss function:
    *   If using Mesh-Proximity: Apply ReLU to the difference between squared distance and $\tau_{pos}$, then sum over all Gaussians [S1].
    *   If using Surface Regularization: Compute the squared L2 norm of the difference between current and initial positions [S2, unofficial].
5.  Multiply by the respective weight ($\lambda_{sr}$ if applicable) and add to the total loss [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Nearest Vertex Search:** Finding $v_i$ requires efficient nearest-neighbor search [S1].
*   **Gradient Flow:** The nearest-vertex search is non-differentiable; gradients are backpropagated through $p_i$ while treating $v_i$ as constant [G].
*   **Storage:** The surface regularization variant requires storing $\mu_{init}$ separately for every Gaussian, increasing memory usage [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def position_regularization_loss(gaussian_positions, mesh_vertices=None, initial_positions=None, tau_pos=0.0):
    """
    Computes position regularization loss.
    
    Args:
        gaussian_positions: List of (x, y, z) tuples for N Gaussians
        mesh_vertices: Optional, list of (x, y, z) tuples for M mesh vertices
        initial_positions: Optional, list of (x, y, z) tuples for N Gaussians
        tau_pos: Threshold for mesh proximity loss
        
    Returns:
        Scalar loss value (L_mesh or L_sr)
    """
    loss = 0.0
    if mesh_vertices is not None:
        # Mesh-Proximity Loss (Human Gaussian Splatting) [S1]
        for p in gaussian_positions:
            min_dist_sq = float('inf')
            for v in mesh_vertices:
                dx, dy, dz = p[0]-v[0], p[1]-v[1], p[2]-v[2]
                dist_sq = dx*dx + dy*dy + dz*dz
                if dist_sq < min_dist_sq:
                    min_dist_sq = dist_sq
            loss += max(0.0, min_dist_sq - tau_pos)
    elif initial_positions is not None:
        # Surface Regularization Loss (Phys3DGS) [S2, unofficial]
        for p, p_init in zip(gaussian_positions, initial_positions):
            dx, dy, dz = p[0]-p_init[0], p[1]-p_init[1], p[2]-p_init[2]
            loss += dx*dx + dy*dy + dz*dz
    return loss
```

## Subtopics

### Canonical Position Regularization
This variant specifically encourages Gaussian positions to stay close to the SMPL mesh to avoid floating artifacts in animatable avatars [S1]. It is chosen when rendering clothed bodies where the distance between skin and clothes is bounded by $\tau_{pos}$ [S1]. It differs from general surface regularization by using a mesh-proximity threshold rather than a fixed initial position constraint [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. [G]
*   **Memory Complexity:** Not given in the sources. [G]
*   **Strengths:** Prevents floating artifacts and dramatic position shifts during training [S1][S2, unofficial].
*   **Limitations:** Requires a reference mesh or initial positions [S1][S2, unofficial].
*   **When to use:** Not given in the sources. [G]
*   **When not to use:** Not given in the sources. [G]

## Background
Originated in Human Gaussian Splatting (2024) for real-time rendering of animatable avatars, where it prevents floating artifacts by constraining Gaussians to the SMPL mesh [S1]. Also used in Phys3DGS (2024) for physically-based inverse rendering to prevent dramatic position shifts of vertices caused by large gradients in HDR space [S2, unofficial].

## Sources
[S1] Human Gaussian Splatting: Real-Time Rendering of Animatable Avatars, https://doi.org/10.1109/cvpr52733.2024.00081, fetched 2026-10-26
[S2] Phys3DGS: Physically-based 3D Gaussian Splatting for Inverse Rendering, https://doi.org/10.48550/arxiv.2409.10335, fetched 2026-10-26 (unofficial)
