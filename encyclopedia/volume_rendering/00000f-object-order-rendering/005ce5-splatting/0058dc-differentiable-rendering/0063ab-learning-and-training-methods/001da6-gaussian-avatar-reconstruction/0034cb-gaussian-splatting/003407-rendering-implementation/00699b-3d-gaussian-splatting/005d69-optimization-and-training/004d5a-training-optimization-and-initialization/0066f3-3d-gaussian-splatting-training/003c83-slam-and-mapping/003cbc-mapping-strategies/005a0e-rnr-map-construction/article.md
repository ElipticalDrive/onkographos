# RNR-Map Construction

Also called: Feature Registration

## Overview
RNR-Map Construction embeds visual information from an RGB-D image into a 2D grid map (RNR-Map) using a pretrained encoder and inverse camera projection [S1]. It differs from sibling methods like [iMAP](11979-imap.md) by operating directly on latent codes to bypass the computationally expensive differentiable rendering step required for alignment, enabling high-frequency mapping [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** An RGB-D image $I_t \in \mathbb{R}^{H \times W \times 4}$, where the first three channels are RGB and the fourth is depth. A relative pose $\Delta p_t = (\Delta x_t, \Delta y_t, \Delta a_t)$ representing the change in $x, y$ position and heading angle $\alpha$ from the previous pose [S1].
*   **Output:** A local RNR-Map $m_t^l \in \mathbb{R}^{U \times V \times D}$ and a local mask $n_t^l \in \mathbb{Z}^{U \times V}$ [S1].

**Data Structures:**
*   **RNR-Map ($m$):** A 2D grid of size $U \times V$. Each cell $m(u,v)$ stores a latent code vector of dimension $D$ ($m(u,v) \in \mathbb{R}^D$) [S1].
*   **Mask ($n$):** A 2D grid of size $U \times V$. Each cell $n(u,v)$ stores an integer count of the number of latent codes averaged into the corresponding map cell [S1].

**Parameters:**
*   $D$: The channel size (dimensionality) of the latent code feature vector [S1].
*   $K$: Camera intrinsic matrix [S1].
*   $\theta_{enc}$: Parameters of the pretrained encoder [S1].
*   $s$: Grid resolution (size of each cell in world units). Not explicitly defined in sources, but implied by the floor division in the mapping formula [S1].

**Steps:**
1.  **Pose Accumulation:** Compute the absolute pose $p_t$ by cumulating relative pose differences from the start pose $p_0 = (0, 0, 0)$ [S1].
2.  **Encoding:** Encode the input image $I_t$ using the pretrained encoder $\theta_{enc}$ to produce a feature map $C_t \in \mathbb{R}^{H \times W \times D}$ [S1].
3.  **Inverse Projection:** For each pixel $(h, w)$ in $C_t$, calculate its 3D world position $[q_x, q_y, q_z]^T$ using the camera intrinsic $K$, the extrinsic matrix derived from $p_t$, and the depth value $d_{h,w}$ from $I_t$ [S1].
4.  **Grid Mapping:** Map the 3D world position to 2D grid coordinates $(u, v)$ using the grid resolution $s$ [S1]:
    $$ u = \left\lfloor \frac{q_x(h,w)}{s} \right\rceil, \quad v = \left\lfloor \frac{q_y(h,w)}{s} \right\rceil $$
    [S1]
5.  **Feature Registration:** Update the map cell $m(u,v)$ and mask cell $n(u,v)$ by averaging the incoming feature $c_{h,w}$ with existing features in that cell [S1]:
    $$ m(u,v) = \frac{1}{n(u,v)} \sum_{c_i \in X_{(u,v)}} c_i, \quad n(u,v) = |X_{(u,v)}| $$
    where $X_{(u,v)}$ is the set of features mapped to cell $(u,v)$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def rnr_map_construction(I_t, delta_p_t, p_prev, K, theta_enc, s):
    """
    Construct/update RNR-Map from a single RGB-D frame.
    
    Args:
        I_t: RGB-D image tensor of shape (H, W, 4).
        delta_p_t: Relative pose (dx, dy, da) from previous step.
        p_prev: Previous absolute pose (x, y, alpha).
        K: Camera intrinsic matrix.
        theta_enc: Pretrained encoder network.
        s: Grid cell size in world units.
        
    Returns:
        m_local: Local RNR-Map tensor of shape (U, V, D).
        n_local: Local mask tensor of shape (U, V).
    """
    # 1. Compute absolute pose
    p_t = p_prev + delta_p_t
    # Note: Exact pose composition (e.g., SE(2) transformation) is not detailed in sources [G]
    
    # 2. Encode image
    C_t = theta_enc(I_t) # Shape: (H, W, D)
    
    # 3. Initialize local map and mask
    # Dimensions U, V depend on the extent of the scene covered by the current frame
    # For implementation, these are typically derived from the bounding box of projected points
    U, V = compute_grid_dimensions(p_t, K, I_t.depth_channel, s)
    m_local = zeros((U, V, D))
    n_local = zeros((U, V), dtype=int)
    
    # 4. Inverse projection and registration
    for h in range(H):
        for w in range(W):
            # Extract depth
            d = I_t[h, w, 3]
            
            # Skip invalid depth
            if d <= 0:
                continue
                
            # Get feature vector
            c = C_t[h, w, :]
            
            # Inverse camera projection to get 3D world position
            # Using K and extrinsic matrix derived from p_t
            # Let R, t be the rotation and translation from p_t
            # Pixel coordinates (u_px, v_px) = (w, h)
            # Homogeneous pixel coordinate
            p_cam = inv(K) @ [w, h, 1]
            # Scale by depth
            p_cam *= d
            # Transform to world frame
            q = R @ p_cam + t
            
            # Map to 2D grid
            # Using floor rounding as per source formula
            u = floor(q[0] / s)
            v = floor(q[1] / s)
            
            # Check bounds
            if 0 <= u < U and 0 <= v < V:
                # Update mask
                n_local[u, v] += 1
                # Update map with running average
                # m(u,v) = (1/n) * sum(c_i)
                # This can be implemented incrementally:
                # m_new = ((n_old * m_old) + c) / (n_old + 1)
                n_old = n_local[u, v] - 1
                if n_old > 0:
                    m_local[u, v] = (n_old * m_local[u, v] + c) / n_local[u, v]
                else:
                    m_local[u, v] = c
                    
    return m_local, n_local
```

## Subtopics

### [Image-Based Localization (Floc)](16555-image-based-localization-floc.md)
Determines an agent's position within an RNR-Map using a single query image by operating directly on latent codes, bypassing rendering.

## Operating characteristics
*   **Time Complexity:** The mapping process operates at 91.9 Hz [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Eliminates the rendering step in mapping, allowing for high-frequency updates suitable for navigation systems [S1].
*   **Limitations:** Requires a pretrained encoder and decoder pair; relies on accurate pose estimation for correct spatial alignment [S1].

## Background
The RNR-Map was introduced by Songhwai Oh et al. in 2023 [S1]. It is designed as a navigation map that supports camera tracking, image-based localization, and image-goal navigation [S1]. The method uses an autoencoder-like framework where the encoder embeds pixel features into the 2D grid map [S1].

## Sources
[S1] Renderable Neural Radiance Map for Visual Navigation, https://doi.org/10.1109/cvpr52729.2023.00878, fetched 2026-10-26
