# Random Pose Perturbation

Also called: Pose Sampling from SE(3)

## Overview
Random Pose Perturbation generates random camera poses within the SE(3) group biased around a reference pose to ensure feature visibility and uniform coverage of the local pose space [S1].

## How it works
**Inputs:** Reference pose $p_{ref}$ (position and orientation), reference axis $\tilde{e}$, angle bound $\theta$, translation bounds [S1].
**Outputs:** Perturbed pose $p_i = p_{ref} + \Delta p_i$, where $\Delta p_i$ consists of a Gibbs vector for orientation and a translation vector [S1].

**Data Structures:**
*   **Gibbs Vector ($q$):** A 3-element vector representing orientation, derived from rotation matrices [S1].
*   **Reference Axis ($\tilde{e}$):** A 3-element unit vector defining the bias direction for rotation sampling [S1].
*   **Perturbation Vector ($\Delta \tilde{e}$):** A 3-element vector added to $\tilde{e}$ to sample the new rotation axis [S1].

**Steps:**
1.  **Sample Rotation Axis:** Sample a perturbation vector $\Delta \tilde{e}$ uniformly on the unit-2 sphere in the neighborhood of the reference axis $\tilde{e}$. The new axis is $\tilde{e}' = \tilde{e} + \Delta \tilde{e}$ [S1].
2.  **Sample Rotation Angle:** Sample an angle $\alpha$ from a non-uniform distribution in the range $[0, \theta]$ for a small $\theta$. This specific distribution yields a uniform random rotation matrix on SO(3) [S1].
3.  **Construct Rotation Matrix:** Construct the rotation matrix $R$ using the axis-angle parameterization with axis $\tilde{e}'$ and angle $\alpha$ [S1].
4.  **Convert to Gibbs Vector:** Convert $R$ to a Gibbs vector $q$ for orientation representation [S1].
5.  **Sample Translation:** Sample translation components $\Delta t$ from a bounded uniform random distribution along each axis. This covers combined translations in transverse and forward motions [S1].
6.  **Combine Pose:** The final perturbed pose is represented by combining the sample translations and Gibbs vectors: $p_i = p_{ref} + \Delta p_i$ [S1].

**Parameters:**
*   $\tilde{e}$: Reference axis around which rotation samples are biased [S1].
*   $\theta$: Small angle range $[0, \theta]$ for rotation sampling [S1].

**Edge Cases and Pitfalls:**
*   **Feature Tracking Failure:** If tracking of at least four features is unsuccessful in the rendered images, the method re-initializes with conservative pose samples [S1].
*   **Minimum Features:** The method requires at least three feature points, though four are preferred for robust tracking [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def sample_random_pose_perturbation(p_ref, e_ref, theta, trans_bounds):
    """
    Generates a random pose perturbation biased around a reference pose.
    
    Args:
        p_ref: Reference pose dictionary with 'position' key (3D array).
        e_ref: Reference axis (3D unit vector).
        theta: Maximum rotation angle bound.
        trans_bounds: Tuple (min_val, max_val) for uniform translation sampling.
        
    Returns:
        perturbed_pose: Dictionary with 'position' and 'orientation_gibbs' keys.
    """
    # 1. Sample rotation axis perturbation uniformly on unit-2 sphere neighborhood
    delta_e = np.random.normal(0, 1, 3)
    e_prime = e_ref + delta_e
    k = e_prime / np.linalg.norm(e_prime)

    # 2. Sample rotation angle from non-uniform distribution in [0, theta]
    # [G] Sources do not specify the exact PDF; standard uniform SO(3) sampling uses 
    # rejection sampling with p(alpha) proportional to sin^2(alpha/2).
    alpha = np.random.uniform(0, theta)

    # 3. Construct Rotation Matrix from Axis-Angle (Rodrigues' formula)
    K = np.array([
        [0, -k[2], k[1]],
        [k[2], 0, -k[0]],
        [-k[1], k[0], 0]
    ])
    R = np.eye(3) + np.sin(alpha) * K + (1 - np.cos(alpha)) * np.dot(K, K)

    # 4. Convert Rotation Matrix to Gibbs Vector (q)
    q = k * np.tan(alpha / 2)

    # 5. Sample Translation
    min_t, max_t = trans_bounds
    delta_t = np.random.uniform(min_t, max_t, 3)

    # 6. Combine Pose
    new_pos = p_ref['position'] + delta_t
    perturbed_pose = {
        'position': new_pos,
        'orientation_gibbs': q
    }
    return perturbed_pose
```

## Subtopics

### Camera and Light Sampling
A variant used in text-to-3D diffusion that samples camera elevation $\phi_{cam}$ from a biased distribution (oversampling poles) with probability 0.5, otherwise from a true uniform-in-area distribution on a half-sphere [S2, unofficial]. The sampled camera position is perturbed by a small uniform offset $U(-0.1, 0.1)^3$ [S2, unofficial]. The “look-at” point is sampled from $N(0, 0.2I)$ and the default “up” vector is perturbed by noise from $N(0, 0.02I)$ [S2, unofficial]. The light direction is sampled from $N(p_{cam}, I)$ and its norm from $U(0.8, 1.5)$, while the camera position norm is sampled from $U(1.0, 1.5)$ [S2, unofficial].

## Operating characteristics
*   **Complexity:** Not given in the sources.
*   **Performance:** Ensures rendered images from adjacent samples have substantial overlap [S1].
*   **Strengths:** Encodes auto and cross-coupled sensitivities in the Jacobian matrix using a minimum number of perturbations [S1]. Ensures feature visibility by biasing samples to a $\delta$-neighborhood of the reference pose [S1].
*   **Limitations:** Requires successful tracking of at least four features; failure triggers re-initialization [S1].

## Background
Developed for differentiable rendering-based pose estimation in proximity operations [S1]. The method relies on the scheme of pose perturbations for Jacobian approximation [S1]. It uses axis-angle parameterization to achieve a uniform random distribution on SO(3) [S1].

## Sources
[S1] Differentiable Rendering for Pose Estimation in Proximity Operations, https://core.ac.uk/download/552738213.pdf, fetched 2026-10-30
[S2] DreamFusion: Text-to-3D using 2D Diffusion, https://doi.org/10.48550/arxiv.2209.14988, fetched 2026-10-27 (unofficial)
