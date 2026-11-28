# L-Tracing

Also called: Light Visibility Estimation on Neural Surfaces by Sphere Tracing, Light Visibility Tracing, Volume Integration for Light Visibility, Volume Rendering Visibility

## Overview
L-Tracing is a method for estimating light visibility on neural implicit surfaces by replacing continuous volumetric integration with binary sphere tracing [S1]. It computes whether a light path is occluded by a solid surface using the Signed Distance Function (SDF), producing binary visibility values (0 or 1) instead of continuous probabilities [S1]. This approach yields sharper shadows and specular highlights compared to volumetric integration, which models translucent occlusion, while achieving approximately 10–13× faster computation [S1].

## How it works
**Inputs:**
*   $x_l$: Light source position (3D vector).
*   $x_s$: Surface point position (3D vector) on the neural implicit surface $\mathcal{S}$.
*   $f$: Neural implicit surface function returning the Signed Distance Function (SDF) value at a query point.
*   $T$: Maximum number of tracing iterations (default 20) [S1].

**Outputs:**
*   $v$: Binary light visibility ($1$ if visible, $0$ if occluded) [S1].

**Data Structures:**
*   No persistent data structures are required; the method is stateless per point-light pair.

**Algorithm Steps:**
1.  **Ray Setup**: Calculate the vector from the light to the surface point: $\mathbf{r} = x_s - x_l$ [S1].
2.  **Range Bound**: Compute the maximum tracing distance (far plane) as the Euclidean norm: $far = \|\mathbf{r}\|_2$ [S1].
3.  **Direction**: Normalize the ray direction: $\mathbf{d} = \mathbf{r} / \|\mathbf{r}\|_2$ [S1].
4.  **Initialization**: Set the accumulated distance $t = 0$ [S1].
5.  **Iteration Loop**: For $i$ from $0$ to $T-1$:
    a.  **Query**: Evaluate the SDF at the current tracing point $pts = x_l + \mathbf{d} \cdot t$: $sd = f(pts)$ [S1].
    b.  **Step**: Advance the ray: $t \leftarrow t + sd$ [S1].
    c.  **Termination Check**: If $t > far$ or $t < 0$, return $v = 1$ (unoccluded) [S1].
6.  **Final Check**: If the loop completes without returning, return $v = 0$ (occluded) [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def l_tracing(x_l, x_s, f, T=20):
    """
    Compute binary light visibility using sphere tracing on an SDF.
    
    Args:
        x_l: Light position (3,)
        x_s: Surface point position (3,)
        f: SDF function f(p) -> float
        T: Max iterations (int)
        
    Returns:
        v: Visibility (1.0 if visible, 0.0 if occluded)
    """
    # 1. Ray Setup
    r = x_s - x_l
    far = torch.norm(r)
    d = r / far
    
    # 2. Initialization
    t = 0.0
    
    # 3. Iteration Loop
    for _ in range(T):
        # Query SDF
        pts = x_l + d * t
        sd = f(pts)
        
        # Step
        t = t + sd
        
        # Termination Check
        if t > far or t < 0:
            return 1.0
            
    # 4. Final Check
    return 0.0
```

**Formulas:**
*   **Convergence Inequality**: For convex surfaces, the error $\epsilon_k$ (distance between tracing point $O_k$ and surface point $Q$) converges linearly:
    $$ 0 < \frac{\epsilon_{k+1}}{\epsilon_k} \leq 1 - \sin\theta $$
    where $\theta$ is the angle between the ray and the surface normal [S1].
*   **Minimum Iterations**: The theoretical lower bound for iterations $T$ to achieve accuracy $\epsilon$ is:
    $$ T \geq \frac{\log \frac{\epsilon}{\epsilon_0}}{\log(1-\sin\theta)} $$
    where $\epsilon_0$ is the initial distance (max range 2) and $\epsilon$ is the target accuracy (0.001) [S1].

**Parameters:**
*   $T$: Maximum iterations. Set to 20 in experiments [S1]. Theoretical minimum for $\epsilon=0.001$ and $\theta_{min}=5^\circ$ is 83, but 20 is sufficient in practice [S1].
*   $\epsilon$: Convergence threshold. Default 0.001 [S1].

**Edge Cases and Pitfalls:**
*   **Tangential Rays**: If the ray is nearly parallel to the surface ($\theta \approx 0$), convergence slows. The theoretical bound assumes $\theta_{min}=5^\circ$ [S1].
*   **Range Violation**: If $t$ exceeds $far$ or becomes negative, the light is considered visible ($v=1$) [S1].
*   **Iteration Limit**: If $T$ iterations are exhausted without $t$ exceeding bounds, the ray is considered trapped/occluded ($v=0$) [S1].

## Operating characteristics
*   **Time Complexity**: Linear in the number of iterations $T$. Each iteration requires one SDF query [S1].
*   **Memory Complexity**: $O(1)$ auxiliary memory per point-light pair [S1].
*   **Performance**: On an NVIDIA RTX 3090, L-Tracing takes 1.07 seconds for 1024 points with 512 lights, compared to 13.36 seconds for Volumetric Integration, a ~13× speedup [S1].
*   **Strengths**: Produces binary, hard shadows and speculars; significantly faster than volumetric integration; respects solid geometry occlusions [S1].
*   **Limitations**: Assumes solid, non-translucent objects; binary visibility may not model soft shadows from translucent materials [S1].
*   **When to Use**: When rendering solid objects with neural implicit surfaces and requiring efficient, sharp shadowing [S1].
*   **When Not to Use**: When modeling translucent materials or soft shadows from volumetric media [S1].

## Background
L-Tracing was introduced by Ziyu Chen et al. in the paper "L-Tracing: Fast Light Visibility Estimation on Neural Surfaces by Sphere Tracing" at ECCV 2022 [S2, unofficial]. It was developed to address the inefficiency and physical inaccuracy of volumetric integration for light visibility in neural implicit surface reconstruction [S1]. The method is used in reflectance factorization frameworks to decompose material and reflectance from multiview images [S1].

## Sources
[S1] L-Tracing: Fast Light Visibility Estimation on Neural Surfaces by Sphere Tracing, https://doi.org/10.1007/978-3-031-19784-0_13, fetched 2026-10-31
[S2] ziyc/L-Tracing, https://github.com/ziyc/L-Tracing, fetched 2026-10-26 (unofficial)
