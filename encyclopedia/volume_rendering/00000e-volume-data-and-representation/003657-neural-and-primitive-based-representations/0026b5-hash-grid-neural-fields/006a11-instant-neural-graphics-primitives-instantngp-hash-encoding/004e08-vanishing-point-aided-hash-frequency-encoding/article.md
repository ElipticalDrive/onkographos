# Vanishing Point Aided Hash-Frequency Encoding

Also called: NeurVPS-ERP

## Overview
Vanishing Point Aided Hash-Frequency Encoding is a neural radiance field encoding scheme that combines multi-resolution hash encoding with deterministic frequency encoding aligned to scene structure via vanishing points [S1]. It solves the problem of poor reconstruction quality from sparse 360° inputs, where standard hash encoding fails due to lack of learnable explicit structure and sensitivity to imperfect scene centering [S1]. Unlike standard hash encoding which relies solely on learnable features, this method adds deterministic frequency features to improve generalizability and robustness without requiring additional depth information or pre-training [S1].

## How it works
The method constructs a final feature vector by concatenating features from a multi-resolution hash grid and a vanishing-point-aligned frequency encoder [S1].

**Inputs and Outputs**
*   **Input:** 3D spatial coordinate $u \in \mathbb{R}^3$ [S1].
*   **Input:** Estimated vanishing points $V = \{v_1, \dots, v_n\}$ derived from input images [S1].
*   **Output:** Feature vector $\gamma(u) \in \mathbb{R}^{kf + 2md}$ [S1].

**Data Structures**
1.  **Multi-resolution Hash Grid:** A set of $k$ levels [S1]. Each level contains a hash table mapping hashed coordinates to learnable feature vectors of size $f$ [S1]. The hash function is a pseudo-random permutation independent for each level [S1].
2.  **Vanishing Points:** A set of $n$ directions, each defined by spherical coordinates $(\lambda_i, \theta_i)$, where $\lambda_i$ is the plane angle (longitude) and $\theta_i$ is the incline (related to elevation) [S1].

**Parameters**
*   $m$: Number of frequency levels in the frequency encoding component [S1].
*   $k$: Number of levels in the multi-resolution hash grid [S1].
*   $f$: Feature size (dimensionality) of each cell in the hash grid [S1].
*   $d$: Dimensionality of the input coordinate $u$ (typically 3 for 3D space) [S1].
*   $\sigma$: Scaling factor for frequency encoding (standard NeRF value, typically logarithmically spaced) [S1].

**Numbered Steps**

1.  **Estimate Vanishing Points:**
    Estimate $n$ vanishing points from the input 2D images [S1]. The source specifies using an adapted NeurVPS network (NeurVPS-ERP) that utilizes conic convolution on Equi-Rectangular Projection (ERP) images to handle omnidirectional data [S1]. Let the estimated vanishing points be $v_i = (\lambda_i, \theta_i)$ for $i=1 \dots n$ [S1].

2.  **Compute Aligned Frequency Features:**
    For each vanishing point $v_i$, compute an angular coordinate $\alpha_i$ for the input $u$ [S1]. This involves transforming $u$ into a coordinate system aligned with the vanishing point direction [S1]. First, convert the 3D point $u$ to spherical coordinates $(\lambda, \phi)$, where $\lambda$ is longitude and $\phi$ is latitude/elevation [S1]. Calculate the elevation $\phi_0$ of the point $p(\lambda, \phi)$ relative to the vanishing point $v(\lambda_i, \theta_i)$ using the formula [S1]:
    $$
    \phi_0(\lambda, \phi) = \arctan\left( \tan(\phi) \sqrt{1 + \tan^2(\lambda - \lambda_i)} - \tan(\theta_i)\tan(\lambda - \lambda_i) \right)
    $$
    The aligned angular coordinate $\alpha_i$ is derived from this geometry [S1]. The source states that points on radial lines originating from the vanishing point share the same $\alpha$ value [S1]. While the exact mapping from $\phi_0$ to $\alpha_i$ is described as modifying the input $u$ to $u_{aligned} = (\alpha_1, \dots, \alpha_n)^\top$, the geometric basis is the calculation of the projected line's properties on the sphere [S1]. The slope of the projected line $L$ is given by [S1]:
    $$
    \frac{d\phi}{d\lambda} = \frac{\tan(\phi_0)\sin(\lambda_i - \lambda) + \tan(\theta_i)\cos(\lambda_i - \lambda)}{\sqrt{(\tan(\phi_0)\cos(\lambda_i - \lambda) - \tan(\theta_i)\sin(\lambda_i - \lambda))^2 + 1}}
    $$
    And the latitude $\phi(\lambda)$ of the projected line is [S1]:
    $$
    \phi(\lambda) = \arctan\left( \frac{\tan(\phi_0) + \tan(\theta_i)\tan(\lambda - \lambda_i)}{\sqrt{1 + \tan^2(\lambda - \lambda_i)}} \right)
    $$
    The value $\alpha_i$ corresponds to the angular deviation or alignment metric derived from these spherical transformations relative to the vanishing point [S1].

3.  **Generate Frequency-Encoded Coordinates:**
    Apply sinusoidal positional encoding to the aligned coordinates $u_{aligned} = (\alpha_1, \dots, \alpha_n)^\top$ [S1]. For each $\alpha_j$, generate $m$ frequency levels [S1]. The frequency-encoded input coordinates $\gamma_{FE}(u) \in \mathbb{R}^{2md}$ are computed as [S1]:
    $$
    \gamma_{FE}(u) = \left[ \dots, \cos(2\pi \sigma \frac{j}{m} \alpha_j), \sin(2\pi \sigma \frac{j}{m} \alpha_j), \dots \right]_{j=0 \dots m-1}
    $$
    Note: The source formula (2) uses $u$, but in this context, $u$ is replaced by the aligned angular coordinates $\alpha$ [S1].

4.  **Hash-Encode Input Coordinates:**
    Hash-encode the original input coordinate $u$ using the $k$-level multi-resolution feature grid [S1]. For each level $l \in \{1, \dots, k\}$: a. Hash the coordinate $u$ to an index in the hash table for level $l$ [S1]. b. Retrieve the learnable feature vector $h_l(u)$ from the hash table [S1]. c. Perform trilinear (or d-linear) interpolation of the features from the surrounding grid cells to obtain the final feature for that level [S1]. Concatenate the features from all $k$ levels to form the hash-encoded feature vector $H(u) \in \mathbb{R}^{kf}$ [S1].

5.  **Concatenate Features:**
    Concatenate the hash-encoded features and the frequency-encoded features to form the final feature vector [S1]:
    $$
    \gamma(u) = [H(u), \gamma_{FE}(u)] \in \mathbb{R}^{kf + 2md}
    $$

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def vanishing_point_aided_hash_frequency_encoding(u, vanishing_points, hash_grids, m, f, sigma):
    """
    Computes the Vanishing Point Aided Hash-Frequency Encoding for a 3D point u.
    
    Args:
        u: numpy array of shape (3,), the 3D spatial coordinate.
        vanishing_points: list of tuples, each containing (lambda_i, theta_i) for each vanishing point.
        hash_grids: list of k hash tables (or grid structures), one for each resolution level.
        m: int, number of frequency levels.
        f: int, feature size per hash grid cell.
        sigma: float, frequency scaling factor.
        
    Returns:
        numpy array of shape (k*f + 2*m*n,), the concatenated feature vector.
    """
    n = len(vanishing_points)
    
    # Step 2: Compute Aligned Frequency Features
    # Convert 3D point u to spherical coordinates (lambda, phi)
    x, y, z = u
    r = math.sqrt(x**2 + y**2 + z**2)
    if r == 0:
        r = 1e-9 # Guard against division by zero
    
    # Longitude lambda
    lambda_u = math.atan2(y, x)
    # Latitude/ elevation phi
    phi_u = math.asin(z / r)
    
    aligned_angles = []
    
    for (lambda_vp, theta_vp) in vanishing_points:
        # Calculate phi_0 relative to vanishing point
        tan_phi = math.tan(phi_u)
        tan_theta_vp = math.tan(theta_vp)
        delta_lambda = lambda_u - lambda_vp
        tan_delta_lambda = math.tan(delta_lambda)
        
        term1 = tan_phi * math.sqrt(1 + tan_delta_lambda**2)
        term2 = tan_theta_vp * tan_delta_lambda
        
        phi_0 = math.atan(term1 - term2)
        
        # The aligned angle alpha is derived from the geometry.
        # The source indicates u_aligned = (alpha_1, ..., alpha_n)^T.
        # While the exact scalar alpha is not explicitly named as a single variable in the text,
        # the geometric alignment is captured by phi_0 and the slope calculations.
        # For implementation, we use phi_0 as the representative aligned angle alpha_i.
        # [G] Using phi_0 as the representative aligned angle alpha_i based on the geometric derivation provided.
        aligned_angles.append(phi_0)
        
    # Generate frequency-encoded input coordinates gamma_FE(u)
    # gamma_FE(u) in R^{2md} -> here d=1 for the scalar alpha, so 2*m*n total dimensions
    freq_features = []
    for alpha in aligned_angles:
        for j in range(m):
            freq = sigma * (j / m)
            freq_features.append(math.cos(2 * math.pi * freq * alpha))
            freq_features.append(math.sin(2 * math.pi * freq * alpha))
            
    freq_features = np.array(freq_features)
    
    # Step 4: Hash-encode the input coordinates
    # Assuming hash_grids is a list of k grids.
    hash_features = []
    for grid in hash_grids:
        # [G] Standard hash grid lookup and trilinear interpolation
        # Converts u to grid coordinates, hashes to table index,
        # fetches 8 corner features, and interpolates.
        level_features = grid.get_features(u) 
        hash_features.extend(level_features)
        
    hash_features = np.array(hash_features)
    
    # Step 5: Concatenate
    final_feature = np.concatenate([hash_features, freq_features])
    
    return final_feature
```

**Edge Cases and Pitfalls**
*   **Sparse Inputs:** Hash encoding fails when input is extremely sparse (e.g., 8 inputs 1 meter apart) [S1]. The method mitigates this by adding deterministic frequency features which do not rely on learnable explicit structure [S1].
*   **Scene Centering:** Scene centering and scaling are crucial for hash encoding performance [S1]. Imperfect positioning in 360° scenes with random camera poses requires the robustness provided by the auxiliary frequency features [S1].
*   **Vanishing Point Estimation:** The method relies on accurate vanishing point estimation from 2D images [S1].

## Operating characteristics
*   **Time Complexity:** The hash encoding part is $O(k)$ per query, where $k$ is the number of resolution levels, assuming $O(1)$ hash table lookups [G]. The frequency encoding part is $O(m \cdot n)$, where $m$ is the number of frequency levels and $n$ is the number of vanishing points [G]. The spherical coordinate transformations are $O(n)$ [G].
*   **Memory Complexity:** The memory usage is dominated by the multi-resolution hash grid, which is $O(k \cdot N_{cells} \cdot f)$, where $N_{cells}$ is the number of cells in the hash grid [G]. The frequency encoding requires negligible memory [G].
*   **Performance:** The method improves reconstruction quality from sparse 360° inputs compared to standard hash encoding [S1]. It does not add overhead to the NeRF network itself or require additional depth information [S1].
*   **Strengths:** Robust to sparse inputs and imperfect scene centering/scaling [S1]. Combines the efficiency of hash encoding with the generalizability of deterministic frequency encoding [S1].
*   **Limitations:** Requires vanishing point estimation as a preprocessing step [S1].

## Background
The method was introduced in the paper "Vanishing Point Aided Hash-Frequency Encoding for Neural Radiance Fields (NeRF) from Sparse 360° Input" [S1]. It builds on Instant-NGP's hash encoding and NeurVPS for vanishing point estimation [S1]. The work addresses the limitations of existing methods for view synthesis from sparse inputs, which may require costly pre-training or additional regularization steps [S1]. It incorporates scene-encoding alignment by estimating vanishing points and modifying the input encoding scheme to ensure alignment with scene geometry [S1].

## Sources
[S1] Vanishing Point Aided Hash-Frequency Encoding for Neural Radiance Fields (NeRF) from Sparse 360°Input, https://doi.org/10.1109/ismar59233.2023.00131, fetched 2026-10-27
