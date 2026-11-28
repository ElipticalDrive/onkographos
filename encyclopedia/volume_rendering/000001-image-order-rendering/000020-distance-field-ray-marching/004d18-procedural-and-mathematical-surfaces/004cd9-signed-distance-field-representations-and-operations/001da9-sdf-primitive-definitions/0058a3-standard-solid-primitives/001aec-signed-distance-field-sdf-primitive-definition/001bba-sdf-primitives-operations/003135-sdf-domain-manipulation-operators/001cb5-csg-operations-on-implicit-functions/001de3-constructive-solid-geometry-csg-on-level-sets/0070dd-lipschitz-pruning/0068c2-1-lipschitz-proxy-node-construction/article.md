# 1-Lipschitz Proxy Node Construction

Also called: 1-Lipschitz Proxy, Proxy Node

## Overview
1-Lipschitz Proxy Node Construction replaces a computationally intensive sub-tree of a Signed Distance Field (SDF) with a simpler proxy volume $V$ and a proxy field $v$, ensuring the resulting composite field $f_e$ is globally 1-Lipschitz continuous [S1]. Unlike standard proxy nodes which may yield discontinuous fields incompatible with smooth Boolean operators, this method maintains continuity, allowing the pruned tree to be used below smooth unions and other operators requiring Lipschitz continuity [S1]. It builds on [Lipschitz Pruning](8623-lipschitz-pruning.md) by providing the specific mechanism for substituting tree branches while preserving the mathematical properties required for safe hierarchical evaluation.

## How it works

### Inputs and Outputs
**Inputs:**
*   $T$: The construction tree of the SDF.
*   $N$: A node in $T$ intended for replacement.
*   $f$: The original signed distance function evaluated by the sub-tree rooted at $N$.
*   $V$: A proxy volume (geometric object) with a computable Euclidean distance function $d(p, V)$.
*   $v$: The proxy field, typically defined as $v(p) = d(p, V)$ or a shifted variant.
*   $\Omega$: The domain of influence, defined as the Minkowski sum $V \oplus B(\delta_r)$, where $B(\delta_r)$ is a ball of radius $\delta_r$ [S1].
*   $\delta_r$: A parameter controlling the size of the transition area between the proxy and the original field [S1].
*   $\lambda_s$: The Lipschitz constant of the interpolating function $s$ used to compute the weighting $\alpha$ [S1].

**Outputs:**
*   A modified tree $T'$ where node $N$ is replaced by a 1-Lipschitz proxy node.
*   The resulting field $f_e(p)$ which is 1-Lipschitz continuous [S1].

### Data Structures
The proxy node stores:
1.  **Proxy Volume $V$**: Defined by its distance function $d(p, V)$. In implementations, this is often a simple primitive (sphere, box) or a convex hull derived from point clouds.
2.  **Proxy Field $v(p)$**: Computed from $V$.
3.  **Transition Radius $\delta_r$**: Scalar value.
4.  **Lipschitz Constant $\lambda$**: Scalar value calculated to ensure the 1-Lipschitz property.
5.  **Interpolation Function $\alpha(p)$**: A function mapping $\mathbb{R}^3 \to [0, 1]$ based on the distance to the boundary of the proxy region.

### Formulas and Parameters

**1. The Interpolating Function $\alpha(p)$**
The weighting function $\alpha(p)$ is computed as a function of the Euclidean distance to the region boundary. Specifically, $\alpha(p) = s\left(\frac{v(p)}{\delta_r}\right)$ or similar distance-based mapping, where $s$ is a smooth step function. The norm of its gradient $\|\nabla \alpha\|$ is equal to the Lipschitz constant $\lambda_s$ [S1].

**2. The Lipschitz Constant $\lambda$**
To guarantee $f_e$ is 1-Lipschitz, $\lambda$ must satisfy:
$$ \lambda = 1 + \frac{\lambda_s \sup_{\Omega} |f - v|}{\delta_r} $$
where $\sup_{\Omega} |f - v|$ is the maximum deviation between the original field $f$ and the proxy field $v$ over the domain $\Omega$ [S1].

**Bounds for $\lambda$:**
*   **Worst Case (No knowledge of $f$):** If $f$ is unknown, the bound is derived via interval analysis. Assuming $f$ represents a distance to a surface within a bounding sphere of radius $r$, and $v$ is bounded by $\delta_r$:
    $$ \sup_{\Omega} |f - v| \le 2r + \delta_r $$
    $$ \lambda \le 1 + \frac{\lambda_s (2r + \delta_r)}{\delta_r} $$
*   **Improved Case (Center inside object):** If the center $c$ of the proxy is inside the object ($f(c) < 0$), the bound tightens:
    $$ \sup_{\Omega} |f - v| \le r + \delta_r $$
    $$ \lambda \le 1 + \frac{\lambda_s (r + \delta_r)}{\delta_r} $$

**3. The Resulting Field $f_e(p)$**
The final field is defined piecewise over three regions [S1]:

$$
f_e(p) =
\begin{cases}
\frac{f(p)}{\lambda} & \text{if } p \in V \\
\frac{1}{\lambda} \left( (1 - \alpha(p)) f(p) + \alpha(p) v(p) \right) & \text{if } p \in \Omega \setminus V \\
(v(p) - \delta_r) + \frac{\delta_r}{\lambda} & \text{otherwise}
\end{cases}
$$

*   **Inside $V$**: The original field is scaled by $\lambda$.
*   **In Transition $\Omega \setminus V$**: Linear interpolation between $f$ and $v$ weighted by $\alpha$, scaled by $\lambda$.
*   **Outside $\Omega$**: The field is determined by the proxy $v$, shifted by $\delta_r$ and scaled [S1].

### Parameters
*   **$\delta_r$**: Controls the transition area size.
    *   *Small $\delta_r$*: Results in a larger Lipschitz bound $\lambda$ (potentially causing larger ray march steps) but provides stronger resilience to side effects from distance field deformation [S1].
    *   *Large $\delta_r$*: Reduces $\lambda$ but increases the region where the proxy distorts the original geometry [S1].
*   **$\lambda_s$**: Lipschitz constant of the interpolation function $s$. Typically determined by the specific smoothstep function used.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def construct_1_lipschitz_proxy_node(
    original_field_func: Callable[[Vector3], float],
    proxy_volume: ProxyVolume,
    delta_r: float,
    lambda_s: float,
    bounding_radius_r: float,
    center_inside_object: bool
) -> ProxyNode:
    """
    Constructs a 1-Lipschitz proxy node.
    
    Args:
        original_field_func: The SDF function f(p) of the sub-tree being replaced.
        proxy_volume: The geometric volume V defining the proxy field v(p).
        delta_r: Transition radius parameter.
        lambda_s: Lipschitz constant of the interpolation function s.
        bounding_radius_r: Radius of the bounding sphere of the original geometry.
        center_inside_object: Boolean indicating if the proxy center is inside the object (f(c) < 0).
        
    Returns:
        A ProxyNode object containing the computed lambda and evaluation logic.
    """
    
    # 1. Calculate the bound for sup |f - v|
    # Based on worst-case interval analysis described in [S1]
    if center_inside_object:
        # Improved bound if center is inside
        sup_diff = bounding_radius_r + delta_r
    else:
        # Worst-case bound if no knowledge of f
        sup_diff = 2 * bounding_radius_r + delta_r
        
    # 2. Calculate the Lipschitz constant lambda
    # Formula: lambda = 1 + (lambda_s * sup_diff) / delta_r
    # Guard against division by zero
    if delta_r <= 0:
        raise ValueError("delta_r must be positive")
        
    lambda_val = 1.0 + (lambda_s * sup_diff) / delta_r
    
    # 3. Define the evaluation function for the proxy node
    def evaluate_proxy(p: Vector3) -> float:
        # Get proxy field value v(p)
        v_p = proxy_volume.distance(p)
        
        # Get original field value f(p)
        # Note: In a real tree, this might be expensive, hence the proxy.
        # However, for the definition of f_e, we assume access to f.
        f_p = original_field_func(p)
        
        # Determine region
        # Region 1: Inside V (p in V)
        # Region 2: In Transition (p in Omega \ V)
        # Region 3: Outside Omega (p not in Omega)
        
        # Check if inside V
        if v_p <= 0:
            return f_p / lambda_val
            
        # Check if inside Omega (V + B(delta_r))
        # v_p <= delta_r implies p is within delta_r of V
        if v_p <= delta_r:
            # Transition region
            # Compute alpha(p)
            # alpha is a function of distance to s(v(p)/delta_r)
            # Assuming s is a standard smoothstep or similar mapping [0,1] -> [0,1]
            # The exact form of s is not specified, but alpha depends on v_p/delta_r
            alpha = compute_alpha(v_p, delta_r, lambda_s)
            
            # Interpolate and scale
            numerator = (1.0 - alpha) * f_p + alpha * v_p
            return numerator / lambda_val
        else:
            # Outside Omega
            return (v_p - delta_r) + (delta_r / lambda_val)

    return ProxyNode(
        lambda=lambda_val,
        evaluate=evaluate_proxy,
        proxy_volume=proxy_volume,
        delta_r=delta_r
    )

def compute_alpha(v_p: float, delta_r: float, lambda_s: float) -> float:
    """
    Computes the weighting factor alpha(p).
    According to [S1], alpha(p) is computed as a function of the Euclidean 
    distance to the region s(v(p)/delta_r).
    
    Since the exact function s is not defined in the notes, we assume a 
    standard linear or smooth interpolation based on the normalized distance.
    A common implementation for such Lipschitz proxies uses a linear blend 
    or a smoothstep. Given ||grad alpha|| = lambda_s, a linear function 
    alpha = v_p / delta_r has gradient 1/delta_r. If lambda_s != 1/delta_r,
    s must be scaled.
    
    For the purpose of this implementation, we assume a linear interpolation 
    normalized by delta_r, adjusted by lambda_s if necessary.
    """
    # Normalized distance in [0, 1]
    t = v_p / delta_r
    
    # Clamp to [0, 1]
    t = max(0.0, min(1.0, t))
    
    # Return alpha. The exact mapping s is implementation-dependent 
    # but must satisfy the Lipschitz constant lambda_s.
    # If s is linear, alpha = t.
    return t
```

### Edge Cases and Pitfalls
*   **Large $\lambda$**: The term $\sup_{\Omega} |f - v|$ can be large, leading to a large $\lambda$. A large $\lambda$ scales down the field values, which can result in larger ray march steps in distance field ray marching, potentially missing thin features or causing visual artifacts [S1].
*   **Choice of $\delta_r$**: There is a trade-off. Smaller $\delta_r$ reduces the deformation of the distance field (better visual fidelity) but increases $\lambda$ (worse numerical stability/ray marching efficiency) [S1].
*   **Discontinuity**: If a standard proxy node (non-1-Lipschitz) is inserted below a smooth operator, the resulting field will be discontinuous, breaking the assumptions of smooth CSG operations [S1].

## Subtopics

### [Lipschitz Proxy Nodes](20313-lipschitz-proxy-nodes.md)
Lipschitz Proxy Nodes (Proxy C) replace a computationally intensive sub-tree with a simpler proxy volume $V$ and field $v$, ensuring the resulting field $f_e$ is globally 1-Lipschitz continuous. Unlike Conservative Proxy Nodes (Proxy P), which yield discontinuous fields and cannot be used below smooth Boolean operators, Proxy C nodes maintain continuity, making them compatible with smooth unions and other operators requiring Lipschitz continuity.

### Automatic Proxy Node Placement via Sphere Carving
An algorithm that uses the Sphere Carving method to generate a point cloud around a surface, extract a convex enclosing object, and use this as a proxy to automatically insert proxy nodes into the signed distance tree [S1]. The algorithm recursively computes a set of convex shapes enclosing sub-trees of the signed distance tree, selects the proxy shape that minimizes the overall cost, and inserts the corresponding proxy node only if it decreases the original cost [S1]. Unlike manual proxy generation, this method automates the creation of proxy nodes using convex shapes derived from surface point clouds, potentially reducing manual effort while maintaining cost efficiency [S1].

## Operating characteristics

**Time Complexity:**
*   **Construction**: Depends on the complexity of generating the proxy volume $V$. For automatic placement via Sphere Carving, it involves recursive computation of convex shapes and cost minimization [S1].
*   **Evaluation**: $O(1)$ per query if the proxy volume distance is $O(1)$. The evaluation involves a few arithmetic operations and a lookup/computation of $\alpha(p)$ [S1].

**Memory Complexity:**
*   Storage for the proxy volume $V$ and the parameter $\lambda$ [S1].

**Performance and Scaling:**
*   **Strengths**: Significantly reduces computational cost by replacing complex sub-trees with simple primitives. Maintains global 1-Lipschitz continuity, enabling safe use in smooth CSG operations [S1].
*   **Limitations**: The quality of the approximation depends on the choice of $V$ and $\delta_r$. Poor choices can lead to large $\lambda$ values, degrading ray marching performance [S1].
*   **When to Use**: When a sub-tree is computationally expensive and the geometry can be well-approximated by a simple proxy volume within a specific region $\Omega$ [S1].
*   **When Not to Use**: When high-fidelity detail is required in the region $\Omega$ and the proxy cannot capture it, or when the resulting $\lambda$ becomes too large for efficient ray marching [S1].

## Background

**Origin:**
The method was introduced by P. Hubert-Brierre, E. Guérin, A. Peytavie, and E. Galin in the paper "Accelerating Signed Distance Functions" [S1].

**History and Development:**
The technique is part of a broader effort to accelerate Signed Distance Field (SDF) evaluations in computer graphics. It addresses the computational bottleneck of complex CSG trees by pruning and replacing expensive branches with simpler proxies, while carefully managing the Lipschitz continuity required for smooth blending operations [S1].

**Usage:**
Used in real-time rendering and offline rendering pipelines where complex implicit surfaces need to be evaluated efficiently [S1].

## Sources

[S1] Accelerating Signed Distance Functions, https://doi.org/10.1111/cgf.70258, fetched 2026-10-26
