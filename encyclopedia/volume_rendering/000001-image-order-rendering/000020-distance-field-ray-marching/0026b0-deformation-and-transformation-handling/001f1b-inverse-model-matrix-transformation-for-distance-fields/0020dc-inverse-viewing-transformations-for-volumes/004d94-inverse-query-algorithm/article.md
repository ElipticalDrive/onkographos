# Inverse Query Algorithm

Also called: Object Manipulator (Inverse Query)

## Overview
The Inverse Query Algorithm is a method for editing learned implicit fields (such as NeRFs) by manipulating object codes in 3D space before 2D projection, thereby avoiding multi-view inconsistencies [S1, unofficial]. It solves the problem of how to edit the density ($\sigma$), color ($c$), and object codes ($o$) of sample points along a query ray such that the rendered novel view reflects the manipulated geometry while handling occlusions and collisions [S1, unofficial]. It sets itself apart from direct editing by mapping sample points to an "inverse space" to determine if they belong to the target object, using projected 2D labels to resolve occlusion inaccuracies [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   Target object code $o_t$: A hard one-hot vector with $H+1$ dimensions, where $H$ is the number of objects and the last dimension represents background/emptiness [S1, unofficial].
    *   Manipulation parameters: Translation $\Delta p = (\Delta x, \Delta y, \Delta z)$, rotation matrix $R_{3\times3}$, and scaling factor $t > 0$ [S1, unofficial].
    *   Sample points $\{p_1, \dots, p_k, \dots, p_K\}$ along a specific query light ray $r$ [S1, unofficial].
    *   Projected pixel object code $\hat{o}$: The object code of the ray before manipulation, obtained from 2D label supervision [S1, unofficial].
*   **Outputs:**
    *   Final pixel color $\bar{c}$ and object code $\bar{o}$ after manipulation [S1, unofficial].

**Data Structures:**
*   Object codes are stored as hard one-hot vectors of size $H+1$ [S1, unofficial].
*   Implicit codes consist of density $\sigma$ (scalar), color $c$ (vector), and object code $o$ (one-hot vector) [S1, unofficial].

**Algorithm Steps:**
1.  **Preprocessing:** Convert all object codes into hard one-hot vectors [S1, unofficial]. Obtain the projected pixel object code $\hat{o}$ for the ray $r$ from the unmanipulated scene [S1, unofficial].
2.  **Ray Loop:** Iterate through each sample point $p_k$ along the ray $r$:
    1.  **Inverse Point Calculation:** Compute the inverse 3D point $p'_k$ using the formula:
        $$ p'_k = \frac{1}{t} R^{-1} (p_k - \Delta p) $$
        where $p_k$ is the current sample point, $t$ is the scale, $R$ is the rotation matrix, and $\Delta p$ is the translation [S1, unofficial].
    2.  **Query Codes:** Query the original implicit codes $\{\sigma_k, c_k, o_k\}$ for $p_k$ and the inverse implicit codes $\{\sigma'_k, c'_k, o'_k\}$ for $p'_k$ using the learned implicit field networks [S1, unofficial].
    3.  **Occlusion Handling:** If the original point belongs to the target object ($o_k = o_t$) but the projected code indicates a different object ($o_k \neq \hat{o}$), update the original object code to the projected code: $o_k \leftarrow \hat{o}$. This corrects inaccuracies in object codes behind surfaces due to insufficient optimization [S1, unofficial].
    4.  **Collision Detection:** If the original point is not the target object ($o_k \neq o_t$), is not background ($o_{H+1} \neq 1$), and the inverse point is the target object ($o'_k = o_t$), a collision is detected. Exit the loop for this ray [S1, unofficial].
    5.  **Code Update:** Determine the new implicit codes $\{\bar{\sigma}_k, \bar{c}_k, \bar{o}_k\}$ for $p_k$ based on the membership of $p_k$ and $p'_k$ relative to $o_t$:
        *   **Non-target to Target:** If $o_k \neq o_t$ and $o'_k = o_t$, set $\{\bar{\sigma}_k, \bar{c}_k, \bar{o}_k\} \leftarrow \{\sigma'_k, c'_k, o'_k\}$ [S1, unofficial].
        *   **Target to Target:** If $o_k = o_t$ and $o'_k = o_t$, set $\{\bar{\sigma}_k, \bar{c}_k, \bar{o}_k\} \leftarrow \{\sigma'_k, c'_k, o'_k\}$ [S1, unofficial].
        *   **Target to Non-target:** If $o_k = o_t$ and $o'_k \neq o_t$, set $\{\bar{\sigma}_k, \bar{c}_k, \bar{o}_k\} \leftarrow \{0, 0, 0\}$ [S1, unofficial].
        *   **Non-target to Non-target:** If $o_k \neq o_t$ and $o'_k \neq o_t$, keep the original codes: $\{\bar{\sigma}_k, \bar{c}_k, \bar{o}_k\} \leftarrow \{\sigma_k, c_k, o_k\}$ [S1, unofficial].
3.  **Volume Rendering:** After processing all $K$ points, compute the final pixel color $\bar{c}$ and object code $\bar{o}$ using the volume rendering equation:
    $$ \bar{c} = \sum_{k=1}^{K} \bar{T}_k \bar{\alpha}_k \bar{c}_k, \quad \bar{o} = \sum_{k=1}^{K} \bar{T}_k \bar{\alpha}_k \bar{o}_k $$
    where the transmittance $\bar{T}_k$ and alpha $\bar{\alpha}_k$ are defined as:
    $$ \bar{T}_k = \exp\left(-\sum_{i=1}^{k-1} \bar{\sigma}_i \delta_i\right), \quad \bar{\alpha}_k = 1 - \exp(-\bar{\sigma}_k \delta_k) $$
    and $\delta_i$ is the distance between adjacent sample points [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def inverse_query_algorithm(p_k_list, o_t, delta_p, R, t, get_implicit_codes, get_projected_code):
    """
    p_k_list: List of 3D points [p1, ..., pK] along the ray
    o_t: Target object one-hot vector (size H+1)
    delta_p: Translation vector (3,)
    R: Rotation matrix (3x3)
    t: Scale factor (float > 0)
    get_implicit_codes: Function returning (sigma, c, o) for a point
    get_projected_code: Function returning the projected object code o_hat for the ray
    """
    # Preprocessing
    o_hat = get_projected_code()
    
    # Storage for edited codes
    edited_sigma = []
    edited_c = []
    edited_o = []
    
    for p_k in p_k_list:
        # 1. Compute inverse point
        # p'_k = (1/t) * R^(-1) * (p_k - delta_p)
        p_prime_k = (1.0 / t) * np.dot(np.linalg.inv(R), (p_k - delta_p))
        
        # 2. Query codes
        sigma_k, c_k, o_k = get_implicit_codes(p_k)
        sigma_prime_k, c_prime_k, o_prime_k = get_implicit_codes(p_prime_k)
        
        # 3. Occlusion handling
        # If original point is target but projected code differs, use projected code
        if np.array_equal(o_k, o_t) and not np.array_equal(o_k, o_hat):
            o_k = o_hat.copy()
            
        # 4. Collision detection
        # If original is not target, not background, and inverse is target -> collision
        is_target = np.array_equal(o_k, o_t)
        is_background = (o_k[-1] == 1) # Last dimension is background
        is_inverse_target = np.array_equal(o_prime_k, o_t)
        
        if not is_target and not is_background and is_inverse_target:
            # Collision detected, exit loop for this ray
            break
            
        # 5. Update codes
        if not is_target and is_inverse_target:
            # Non-target to Target
            new_sigma = sigma_prime_k
            new_c = c_prime_k
            new_o = o_prime_k
        elif is_target and is_inverse_target:
            # Target to Target
            new_sigma = sigma_prime_k
            new_c = c_prime_k
            new_o = o_prime_k
        elif is_target and not is_inverse_target:
            # Target to Non-target
            new_sigma = 0.0
            new_c = np.zeros_like(c_k)
            new_o = np.zeros_like(o_k)
        else:
            # Non-target to Non-target
            new_sigma = sigma_k
            new_c = c_k
            new_o = o_k
            
        edited_sigma.append(new_sigma)
        edited_c.append(new_c)
        edited_o.append(new_o)
        
    # 6. Volume Rendering
    # Calculate deltas (distances between points)
    # Assuming p_k_list is ordered from near to far
    deltas = []
    for i in range(len(p_k_list)):
        if i == 0:
            # Distance from camera origin to first point? 
            # Standard NeRF usually assumes fixed intervals or computes from points.
            # Here we assume delta_i is known or computed from p_k_list[i] - p_k_list[i-1]
            # For the first point, we need a starting distance. 
            # The source does not specify the first delta. [G] Assuming delta_1 is provided or computed from ray origin.
            # To be self-contained, we assume deltas are passed or computed externally.
            # However, the formula uses delta_i. We will assume deltas are available.
            pass 
        else:
            deltas.append(np.linalg.norm(p_k_list[i] - p_k_list[i-1]))
            
    # Note: The source formula uses delta_i for the i-th term in the sum.
    # We need to handle the first delta. In standard volume rendering, delta_1 is distance from camera to p_1.
    # Since this is not explicitly given, we assume deltas are precomputed for the ray.
    # If not, this step requires external context. [G] We assume 'deltas' is a list of length K.
    
    # Compute transmittance and alpha
    K = len(edited_sigma)
    T_bar = 1.0
    final_color = np.zeros_like(edited_c[0]) if K > 0 else np.zeros(3)
    final_o = np.zeros_like(edited_o[0]) if K > 0 else np.zeros(H+1)
    
    for k in range(K):
        sigma = edited_sigma[k]
        c = edited_c[k]
        o = edited_o[k]
        delta = deltas[k] # Assuming deltas[0] is distance to p_1, etc.
        
        alpha = 1.0 - np.exp(-sigma * delta)
        T_bar_k = T_bar * (1.0 - alpha) # T_k = exp(-sum_{i=1}^{k-1} sigma_i delta_i)
        
        final_color += T_bar_k * alpha * c
        final_o += T_bar_k * alpha * o
        
        T_bar = T_bar_k
        
    return final_color, final_o
```

**Parameters:**
*   $H$: Number of objects. The object code dimension is $H+1$ [S1, unofficial].
*   $\Delta d$: A hyper-parameter for the object field loss, set to 0.05 meters in experiments [S1, unofficial].
*   $t$: Scaling factor, $t > 0$ [S1, unofficial].

**Pitfalls:**
*   **Collisions:** Objects may collide during manipulation. This is addressed by the emptiness score in the last dimension of the object code $o$ [S1, unofficial].
*   **Occlusion Inaccuracy:** Object codes behind surface points may be inaccurate due to insufficient optimization. This is mitigated by using the projected object code $\hat{o}$, which is more accurate due to 2D label supervision [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of sample points $K$ per ray, as it requires two implicit field queries per point (original and inverse) and a loop over $K$ points [S1, unofficial].
*   **Memory Complexity:** Linear with respect to $K$ to store the edited codes for volume rendering [S1, unofficial].
*   **Strengths:** Handles occlusions and collisions explicitly; allows editing in 3D space before projection to maintain consistency [S1, unofficial].
*   **Limitations:** Requires pre-computation of projected codes $\hat{o}$; relies on the accuracy of the inverse transformation and the learned implicit fields [S1, unofficial].
*   **When to use:** When manipulating specific objects in a decomposed 3D scene represented by implicit fields, especially when occlusion and collision handling are critical [S1, unofficial].
*   **When not to use:** For scenes without object decomposition or when real-time performance is critical and $K$ is very large [S1, unofficial].

## Background
The Inverse Query Algorithm was introduced in the paper "DM-NeRF: 3D Scene Geometry Decomposition and Manipulation from 2D Images" by [S1, unofficial]. It addresses the challenges of editing implicit fields by leveraging 2D supervision to correct 3D ambiguities. The method is part of a pipeline that decomposes 3D scenes into object fields and a background field, allowing for individual object manipulation [S1, unofficial].

## Sources
[S1] DM-NeRF: 3D Scene Geometry Decomposition and Manipulation from 2D Images, https://arxiv.org/abs/2208.07227v2, fetched 2026-10-26 (unofficial)
