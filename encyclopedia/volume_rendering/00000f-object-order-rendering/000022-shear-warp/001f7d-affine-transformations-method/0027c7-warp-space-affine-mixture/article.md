# Warp-Space Affine Mixture

Also called: Affine mixture warp model

## Overview
Warp-Space Affine Mixture is a warping technique for neural volume rendering that computes affine mixture weights in the warped coordinate space rather than world space. It solves the problem of modeling complex, non-linear deformations (such as hair swing or smoke) by combining multiple local affine transformations. It sets itself apart from convolutional warp models and world-space mixture models by ensuring that the spatial weighting volumes are evaluated after the point has been transformed, which prevents the warps from getting "stuck" and improves generalization on dynamic scenes [S1].

## How it works
The method constructs an inverse warp field $\mathbf{W}^{-1}(\mathbf{x})$ as a spatial mixture of $N$ affine transformations. The core innovation is that the mixture weights $a_i(\mathbf{x})$ are determined by sampling the weight volumes $w_i$ at the *warped* location, not the original location.

**Inputs:**
*   $\mathbf{x}$: A 3D point in the output (warped) space.
*   $z$: A latent encoding vector (e.g., from an encoder network).
*   $\{R_i, s_i, t_i\}_{i=1}^N$: Parameters for $N$ affine warps.
*   $\{w_i\}_{i=1}^N$: Weight volumes for $N$ warps.
*   $\{R_g, s_g, t_g\}$: Global warp parameters.

**Outputs:**
*   $\mathbf{x}'$: The corresponding point in the input (source) space, i.e., $\mathbf{x}' = \mathbf{W}^{-1}(\mathbf{x})$.

**Data Structures:**
*   **Weight Volumes ($w_i$):** 3D voxel grids of resolution $32^3$. These are stored as dense arrays. Values are non-negative.
*   **Affine Parameters:**
    *   $R_i$: $3 \times 3$ rotation matrix.
    *   $s_i$: $3 \times 1$ scaling vector.
    *   $t_i$: $3 \times 1$ translation vector.
*   **Global Warp Parameters:** $\{R_g, s_g, t_g\}$, produced by an MLP.

**Parameters:**
*   Number of warps $N$: 16 [S1].
*   Warp field resolution: $32^3$ voxels [S1].

**Steps:**

1.  **Encode:** Obtain latent vector $z$ from the input image sequence or frame index.
2.  **Compute Local Affine Parameters:**
    *   Pass $z$ through two fully-connected layers.
    *   For each warp $i \in \{1, \dots, N\}$:
        *   Extract rotation quaternion $q_i$. Normalize $q_i$ and convert to rotation matrix $R_i$.
        *   Extract scaling vector $s_i$ and translation vector $t_i$.
        *   Extract raw weight values for the volume $w_i$. Apply $\exp(\cdot)$ to ensure non-negative weights.
3.  **Compute Global Warp:**
    *   Pass $z$ through an MLP to produce global parameters $\{R_g, s_g, t_g\}$.
4.  **Apply Global Warp:**
    *   Transform the input point $\mathbf{x}$ by the global warp to get $\mathbf{x}_{global}$:
        $$ \mathbf{x}_{global} = R_g (s_g \circ (\mathbf{x} - t_g)) $$
5.  **Compute Warped Weights:**
    *   For each warp $i$:
        *   Apply the local affine transformation $A_i$ to $\mathbf{x}_{global}$ to get the warped coordinate $\mathbf{x}_i$:
            $$ \mathbf{x}_i = R_i (s_i \circ (\mathbf{x}_{global} - t_i)) $$
        *   Sample the weight volume $w_i$ at $\mathbf{x}_i$.
        *   **Pitfall Handling:** If $\mathbf{x}_i$ is outside the bounds of the weight volume $w_i$, clamp the sample to the surface of the volume (or boundary) to prevent gradients from vanishing or warps from getting stuck [S1]. Let this value be $w_i(\mathbf{x}_i)$.
6.  **Compute Mixture Weights:**
    *   Calculate the normalized weight $a_i(\mathbf{x})$ for each warp:
        $$ a_i(\mathbf{x}) = \frac{w_i(\mathbf{x}_i)}{\sum_{j=1}^{N} w_j(\mathbf{x}_j)} $$
7.  **Compute Final Warped Point:**
    *   Sum the affine transformations weighted by $a_i(\mathbf{x})$:
        $$ \mathbf{x}' = \sum_{i=1}^{N} A_i(\mathbf{x}_{global}) a_i(\mathbf{x}) $$
        Where $A_i(\mathbf{x}_{global}) = R_i (s_i \circ (\mathbf{x}_{global} - t_i))$.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def quaternion_to_rotation_matrix(q):
    """Convert a 4D quaternion vector to a 3x3 rotation matrix."""
    # q is [w, x, y, z]
    w, x, y, z = q[0], q[1], q[2], q[3]
    return np.array([
        [1 - 2*y*y - 2*z*z, 2*x*y - 2*z*w, 2*x*z + 2*y*w],
        [2*x*y + 2*z*w, 1 - 2*x*x - 2*z*z, 2*y*z - 2*x*w],
        [2*x*z - 2*y*w, 2*y*z + 2*x*w, 1 - 2*x*x - 2*y*y]
    ])

def clamp_to_surface(volume, point):
    """
    If point is outside volume bounds, clamp to the nearest surface voxel.
    Assumes volume is defined on grid [0, 1]^3 normalized.
    """
    # [G] Implementation detail: In practice, this involves finding the nearest
    # valid voxel index or interpolating boundary values.
    # For the purpose of this method description, we assume a function
    # that returns the value at the clamped coordinate.
    return volume[point] # Placeholder for actual clamping logic

def warp_space_affine_mixture(x, z, params, weight_volumes):
    """
    x: 3D point in warped space (3,)
    z: latent vector
    params: dict containing 'local_params' (list of N dicts with R, s, t)
            and 'global_params' (dict with Rg, sg, tg)
    weight_volumes: list of N 3D numpy arrays (32, 32, 32)
    """
    N = len(params['local_params'])
    
    # 1. Compute Global Warp
    Rg = params['global_params']['Rg']
    sg = params['global_params']['sg']
    tg = params['global_params']['tg']
    
    # x_global = Rg * (sg * (x - tg))
    x_global = Rg @ (sg * (x - tg))
    
    # 2. Compute Local Warps and Weights
    warped_points = []
    raw_weights = []
    
    for i in range(N):
        Ri = params['local_params'][i]['R']
        si = params['local_params'][i]['s']
        ti = params['local_params'][i]['t']
        
        # Apply local affine to x_global
        # x_i = Ri * (si * (x_global - ti))
        x_i = Ri @ (si * (x_global - ti))
        warped_points.append(x_i)
        
        # 3. Sample Weight Volume
        # [G] Assuming trilinear interpolation is used for sampling
        # and coordinates are normalized to [0, 1] or similar grid space.
        # If x_i is out of bounds, clamp to surface.
        w_i = weight_volumes[i][x_i] # Simplified access; requires interpolation/clamping
        
        # Apply exp to ensure non-negative weights (if raw weights were not pre-exp'd)
        # The source says exp is applied before outputting values, so we assume
        # weight_volumes[i] already contains exp(raw_weights).
        raw_weights.append(w_i)
        
    # 4. Normalize Weights
    sum_weights = sum(raw_weights)
    if sum_weights == 0:
        # [G] Fallback if all weights are zero
        a_i = [1.0 / N] * N
    else:
        a_i = [w / sum_weights for w in raw_weights]
        
    # 5. Compute Final Point
    x_prime = np.zeros(3)
    for i in range(N):
        Ri = params['local_params'][i]['R']
        si = params['local_params'][i]['s']
        ti = params['local_params'][i]['t']
        
        # A_i(x_global) = Ri * (si * (x_global - ti))
        A_i_x_global = Ri @ (si * (x_global - ti))
        
        x_prime += A_i_x_global * a_i[i]
        
    return x_prime
```

## Subtopics

### Mixture of Affine Warps
This variant refers to the specific architectural choice of using a spatial mixture of affine warps as the basic building block of the warp field, as opposed to freely-varying template sample points or convolutional fields. It is chosen to avoid overfitting and poor generalization to novel views [S1]. The implementation uses 2 fully-connected layers after encoding $z$ to produce transformation parameters. Rotation is handled by producing a quaternion vector, normalizing it, and transforming it into a rotation matrix. Weights are computed by applying $\exp(\cdot)$ to the output of the fully-connected layers for the weighting volume $w_i$ to ensure non-negative weights. A global warp $\{R_g, s_g, t_g\}$ is learned via an MLP and applied to $\mathbf{x}$ before the warping field. The number of warps is set to 16, and the warp field resolution is $32^3$ to provide smoothness via trilinear interpolation [S1]. A key pitfall is that samples outside the weighting volume $w_i$ must be clamped to the surface to prevent warps from getting "stuck" early in training [S1].

## Operating characteristics
*   **Time Complexity:** The warping step involves $N$ affine transformations and $N$ volume lookups. With $N=16$, this is constant time relative to scene complexity but linear in the number of warps. Trilinear interpolation is used for volume sampling.
*   **Memory Complexity:** Requires storing $N$ weight volumes of size $32^3$. For $N=16$, this is $16 \times 32^3$ voxels.
*   **Performance:** Outperforms models with no warping, convolutional warps (transposed convolutions or trilinear upsampling + convolution), and no-warp affine mixture (world-space weighting) [S1]. It is particularly effective on dynamic scenes with complex deformations like hair swing, where convolutional warps fail [S1].
*   **Strengths:** Better generalization to novel views compared to template-based methods; robust handling of non-linear bending; smooth interpolation due to low-resolution weight volumes.
*   **Limitations:** Requires learning additional parameters (global warp, local warp parameters, weight volumes); sensitive to initialization (requires clamping to avoid getting stuck).
*   **When to use:** When rendering dynamic volumetric data with significant non-linear deformations (e.g., hair, smoke) and view-dependent appearance is also modeled.
*   **When not to use:** For static volumes or simple rigid motions where simpler warping or no warping suffices.

## Background
The Warp-Space Affine Mixture method was introduced in the paper "Neural Volumes: Learning Dynamic Renderable Volumes from Images" [S1]. It is part of the broader Neural Volumes framework, which learns dynamic 3D scenes from a sequence of images. The method was developed to address the limitations of previous warping techniques, such as convolutional warps, which struggled with complex deformations like hair swing. The approach is inspired by linear blend skinning [Lewis et al. 2000], extending it to volumetric space by computing mixture weights in the warped space.

## Sources
[S1] Neural Volumes: Learning Dynamic Renderable Volumes from Images, https://arxiv.org/abs/1906.07751, fetched 2026-10-26
