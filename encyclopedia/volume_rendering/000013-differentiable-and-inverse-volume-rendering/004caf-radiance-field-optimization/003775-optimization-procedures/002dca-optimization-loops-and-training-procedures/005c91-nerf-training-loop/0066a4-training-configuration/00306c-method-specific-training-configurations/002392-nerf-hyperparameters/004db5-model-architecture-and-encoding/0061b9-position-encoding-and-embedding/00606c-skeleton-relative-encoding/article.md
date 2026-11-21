# Skeleton-Relative Encoding

Also called: A-NeRF Skeleton-Relative Encoding, Relative Encoding, Skeleton-Relative Input Encoding

## Overview
Skeleton-Relative Encoding transforms 3D query positions and view directions into local bone-relative coordinate systems before processing them through a neural radiance field, explicitly incorporating domain knowledge of human body part linkage [S1, unofficial]. Unlike standard NeRF which treats space as continuous and unstructured, this method reparameterizes the input space to respect the piece-wise rigidity of articulated motion, making it suitable for dynamic human avatars where global coordinate systems are insufficient [S1, unofficial]. It sets itself apart from siblings like [Mip-NeRF 360 MLP Architecture](14098-mip-nerf-360-mlp-architecture.md) by replacing global positional encoding with bone-centric relative metrics, and from [SUDS](14964-suds.md) by relying on explicit skeleton transforms rather than implicit deformation fields.

## How it works

### Inputs and Outputs
*   **Inputs:**
    *   World-space query position $q \in \mathbb{R}^3$.
    *   World-space view direction $d \in \mathbb{R}^3$.
    *   Skeleton pose parameters $\theta_k$ for frame $k$, defining bone transformations $T(\theta_k, m)$ for each bone $m$.
*   **Outputs:**
    *   Combined encoding vector $e_k \in \mathbb{R}^D$, which serves as the input to the density/color MLP $F_\phi$.

### Data Structures
*   **Skeleton Representation:** A set of $M$ bones (typically $M=24$ for a standard human skeleton, indices $1$ to $24$) [S1, unofficial].
*   **Bone Transformation:** $T(\theta_k, m)$ is a $4 \times 4$ homogeneous transformation matrix mapping local bone coordinates to world coordinates for bone $m$ at pose $\theta_k$ [S1, unofficial].
*   **Encoding Components:**
    *   Relative Distance $\tilde{v}_k \in \mathbb{R}^M$: Scalar distance from $q$ to each bone.
    *   Relative Direction $\tilde{r}_k \in \mathbb{R}^{3M}$: Unit vector from $q$ to each bone.
    *   Relative Ray Direction $\tilde{d}_k \in \mathbb{R}^{3M}$: View direction rotated to local bone frame.

### Steps and Formulas

1.  **Compute Bone-relative Position:**
    For each bone $m \in \{1, \dots, M\}$, transform the world query $q$ into the local coordinate system of bone $m$ using the inverse transformation:
    $$ \tilde{q}_{k,m} = T^{-1}(\theta_k, m) q $$
    where $\tilde{q}_{k,m} \in \mathbb{R}^3$ [S1, unofficial].

2.  **Compute Relative Distance:**
    Calculate the L2 norm of the bone-relative position to obtain the radial distance:
    $$ \tilde{v}_{k,m} = \| \tilde{q}_{k,m} \|_2 \in \mathbb{R} $$
    The vector of distances for all bones is $\tilde{v}_k = [\tilde{v}_{k,1}, \dots, \tilde{v}_{k,M}]$ [S1, unofficial].

3.  **Compute Relative Direction:**
    Normalize the bone-relative position to capture orientation information:
    $$ \tilde{r}_{k,m} = \frac{\tilde{q}_{k,m}}{\| \tilde{q}_{k,m} \|_2} \in \mathbb{R}^3 $$
    The vector of directions is $\tilde{r}_k = [\tilde{r}_{k,1}, \dots, \tilde{r}_{k,M}]$ [S1, unofficial].

4.  **Compute Relative Ray Direction:**
    Rotate the incoming world-space view direction $d$ by the rotational part of the inverse bone-to-world transformation:
    $$ \tilde{d}_{k,m} = [T^{-1}(\theta_k, m)]_{3\times3} d \in \mathbb{R}^3 $$
    where $[T^{-1}(\theta_k, m)]_{3\times3}$ is the upper-left $3 \times 3$ rotation submatrix [S1, unofficial].
    The vector of ray directions is $\tilde{d}_k = [\tilde{d}_{k,1}, \dots, \tilde{d}_{k,M}]$ [S1, unofficial].

5.  **Apply Cutoff Function:**
    To prevent distant bones from influencing the prediction, apply a sigmoid-based cutoff function $h(\cdot)$ to the distance and ray direction encodings:
    $$ h(\tilde{v}_{k,m}) = 1 - S(\tau (\tilde{v}_{k,m} - t)) $$
    where $S(\cdot)$ is the sigmoid function, $t$ is the cutoff threshold, and $\tau$ controls the sharpness of the cutoff [S1, unofficial].

6.  **Construct Final Encoding:**
    Combine the components into the final input vector $e_k$. Positional Encoding $\Gamma(\cdot)$ is applied to the relative distance and relative ray direction, but **not** to the relative direction $\tilde{r}_k$ [S1, unofficial].
    $$ e_k = [h(\tilde{v}_k) \odot \Gamma(\tilde{v}_k), \quad \tilde{r}_k, \quad h(\tilde{v}_k) \odot \Gamma(\tilde{d}_k)] $$
    Here, $\odot$ denotes element-wise multiplication. $\Gamma(\cdot)$ is the hierarchical sinusoidal positional encoding defined in the parent article [Model Architecture and Encoding](13550-model-architecture-and-encoding.md) [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def skeleton_relative_encoding(q, d, bone_transforms_inv, t=1.0, tau=10.0, L=10):
    """
    Computes the skeleton-relative encoding for a query point and view direction.
    
    Args:
        q: np.array, shape (3,), world-space query position.
        d: np.array, shape (3,), world-space view direction.
        bone_transforms_inv: list of np.array, shape (4, 4), inverse bone-to-world transforms for M bones.
        t: float, cutoff threshold for the sigmoid function.
        tau: float, sharpness of the cutoff.
        L: int, number of frequency bands for positional encoding.
        
    Returns:
        e_k: np.array, the combined encoding vector.
    """
    M = len(bone_transforms_inv)
    
    # Lists to store components for each bone
    rel_distances = []
    rel_directions = []
    rel_ray_directions = []
    
    for m in range(M):
        # Step 1: Bone-relative position
        # T_inv is 4x4, q is (3,). We treat q as (3,1) for matrix mult or handle manually.
        # T_inv @ [q, 1] gives [x, y, z, w]. We take the first 3 components.
        q_homogeneous = np.append(q, 1.0)
        q_local_homogeneous = bone_transforms_inv[m] @ q_homogeneous
        q_local = q_local_homogeneous[:3]
        
        # Step 2: Relative Distance
        v_m = np.linalg.norm(q_local)
        rel_distances.append(v_m)
        
        # Step 3: Relative Direction
        # Avoid division by zero if q is exactly at the bone origin
        if v_m < 1e-8:
            r_m = np.zeros(3)
        else:
            r_m = q_local / v_m
        rel_directions.append(r_m)
        
        # Step 4: Relative Ray Direction
        # Extract 3x3 rotation part of the inverse transform
        R_inv = bone_transforms_inv[m][:3, :3]
        d_local = R_inv @ d
        rel_ray_directions.append(d_local)
        
    # Convert to arrays
    v_k = np.array(rel_distances)          # Shape (M,)
    r_k = np.array(rel_directions)         # Shape (M, 3)
    d_k = np.array(rel_ray_directions)     # Shape (M, 3)
    
    # Step 5: Cutoff Function
    # Sigmoid: 1 / (1 + exp(-x))
    sigmoid_input = tau * (v_k - t)
    h = 1.0 - (1.0 / (1.0 + np.exp(-sigmoid_input)))
    
    # Step 6: Positional Encoding and Combination
    # Apply PE to distances
    # Gamma(v) returns a vector of size 2*L for each scalar v
    # We apply it element-wise to the array v_k
    v_pe = positional_encoding(v_k, L)   # Shape (M, 2*L)
    
    # Apply PE to ray directions
    # Gamma(d) is applied to each component of d_k? 
    # The source says Gamma(d_k). In standard NeRF, PE is applied to each scalar component.
    # d_k is (M, 3). We flatten to (3*M,) or apply per component.
    # Standard PE: for scalar p, returns (sin(2^0 pi p), cos(2^0 pi p), ...)
    # So for vector d_k, we apply PE to each of the 3 components for each bone.
    d_pe_flat = positional_encoding(d_k.flatten(), L) # Shape (3*M, 2*L)
    d_pe = d_pe_flat.reshape(M, 3 * 2 * L) # Shape (M, 6*L)
    
    # Apply cutoff h to the PE outputs
    # h is (M,). v_pe is (M, 2*L). Broadcast h to (M, 1)
    v_pe_cropped = v_pe * h[:, np.newaxis]
    d_pe_cropped = d_pe * h[:, np.newaxis]
    
    # Concatenate: [h(v)Gamma(v), r, h(d)Gamma(d)]
    # r is (M, 3)
    # v_pe_cropped is (M, 2*L)
    # d_pe_cropped is (M, 6*L)
    
    e_k = np.concatenate([v_pe_cropped, r_k, d_pe_cropped], axis=1)
    
    return e_k

def positional_encoding(values, L):
    """
    Applies hierarchical sinusoidal positional encoding to a 1D array of values.
    
    Args:
        values: np.array, shape (N,), scalar values to encode.
        L: int, number of frequency bands.
        
    Returns:
        encoded: np.array, shape (N, 2*L).
    """
    N = len(values)
    encoded = np.zeros((N, 2 * L))
    for i in range(L):
        freq = 2**i * np.pi
        encoded[:, 2*i] = np.sin(values * freq)
        encoded[:, 2*i+1] = np.cos(values * freq)
    return encoded
```

### Parameters
*   **Number of Bones ($M$):** 24, corresponding to indices 1 to 24 of the human skeleton [S1, unofficial].
*   **Cutoff Threshold ($t$):** Not given in the sources.
*   **Sharpness ($\tau$):** Not given in the sources.
*   **Frequency Bands ($L$):** Not given in the sources (inherited from parent [Model Architecture and Encoding](13550-model-architecture-and-encoding.md)).

### Edge Cases and Pitfalls
*   **Dimensionality Explosion:** Using bone-relative *position* ($\tilde{q}_{k,m}$) instead of relative *distance* ($\tilde{v}_{k,m}$) increases the input dimensionality by an order of magnitude, which can hinder learning [S1, unofficial]. The method prefers relative distance for lower dimensionality and better reconstruction accuracy.
*   **Relative Direction Encoding:** The relative direction $\tilde{r}_k$ is passed directly into the final encoding $e_k$ without subsequent positional encoding (PE). Applying PE to direction vectors is not beneficial in this context [S1, unofficial].
*   **Ambiguity:** While bone-relative position removes ambiguities of assigning a query to a single part, the radial distance encoding naturally captures spherically shaped limbs and is robust to the ill-posed association to a single part [S1, unofficial].

## Subtopics

### Bone-relative Position Encoding
Maps the query $q$ to local coordinates of each bone. Sets itself apart by preserving full 3D local structure but suffers from high dimensionality. Choose when local geometric detail per bone is critical and computational budget allows. [S1, unofficial]

### Relative Distance Encoding
Computes the L2 norm from $q$ to each bone. Sets itself apart by being lower-dimensional and capturing radial symmetry of limbs. This is the preferred encoding in the final A-NeRF model. [S1, unofficial]

### Relative Direction Encoding
Normalizes the bone-relative position vector. Sets itself apart by providing orientation information without PE. Always used in conjunction with distance encoding. [S1, unofficial]

### Relative Ray Direction Encoding
Rotates the view direction into the local bone frame. Sets itself apart by enabling view-dependent appearance modeling relative to the body part. [S1, unofficial]

### Cutoff Function
A sigmoid-based weighting mechanism. Sets itself apart by localizing the influence of each bone, preventing distant bones from affecting the prediction. [S1, unofficial]

## Operating characteristics

*   **Time Complexity:** $O(M \cdot L)$ per query point, where $M$ is the number of bones and $L$ is the number of frequency bands. The transformation $T^{-1}$ is $O(1)$ per bone.
*   **Memory Complexity:** $O(M \cdot L)$ for the encoding vector. The skeleton transforms must be stored or computed on the fly.
*   **Strengths:**
    *   Incorporates explicit domain knowledge of human articulation.
    *   Invariant to global shift and rotation of the person.
    *   Preserves piece-wise rigidity of articulated motion.
    *   Lower dimensionality compared to full bone-relative position encoding.
*   **Limitations:**
    *   Requires a skeleton pose estimator for initialization and runtime.
    *   Performance depends on the accuracy of the skeleton tracking.
    *   Does not model complex soft-tissue deformation (muscle bulging) as explicitly as parametric surface models, though it approximates it via the neural field.
*   **When to use:** Dynamic human avatar rendering where pose information is available or can be estimated.
*   **When not to use:** Static scenes, scenes without articulated structures, or when skeleton tracking is unreliable.

## Background

Skeleton-Relative Encoding was introduced in **A-NeRF: Articulated Neural Radiance Fields for Learning Human Shape, Appearance, and Pose** by Gkioxari et al. (2021) [S1, unofficial]. The method was developed to address the limitations of standard NeRF in modeling dynamic humans, where global coordinates fail to capture the rigid motion of body parts. It inverts the approach of parametric models like SMPL: instead of deforming a surface, it deforms the query space. The work builds on the foundational NeRF architecture [Model Architecture and Encoding](13550-model-architecture-and-encoding.md) and integrates it with articulated skeleton data.

## Sources

[S1] A-NeRF: Articulated Neural Radiance Fields for Learning Human Shape, Appearance, and Pose, https://doi.org/10.48550/arxiv.2102.06199, fetched 2026-10-30 (unofficial)
