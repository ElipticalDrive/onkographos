# Fabric Tube Rendering


## Overview
Fabric Tube Rendering is a post-processing technique that interprets a composited image of a tensor field as streamtubes on the surface without creating actual 3D geometry [S1, book]. It uses binormals derived from eigenvectors and image-plane normals to determine tube position and width, creating a tube-like effect with proper spatial impression [S1, book]. It sets itself apart from [Stream Tube Generation](2707-stream-tube-generation.md) by avoiding explicit geometric construction and integration along the field, instead relying on local image-space analysis to compute shading normals [S1, book].

## How it works
**Inputs and Outputs:**
- **Input:** A composited image $R(x,y)$ (or $G(x,y)$) representing the fabric structure, the eigenvectors $v'_{\lambda_0}, v'_{\lambda_1}$ in image space, and a threshold $\varepsilon$ [S1, book].
- **Output:** A shaded image with a fabric-like spatial impression using Phong shading [S1, book].

**Data Structures:**
- Image pixels at coordinates $(x,y)$ [S1, book].
- Eigenvectors $v'_{\lambda_i}$ for $i \in \{0, 1\}$ at each pixel [S1, book].
- Image plane normal $n_{plane} = (0,0,1)^T$ [S1, book].

**Steps:**

1.  **Calculate Binormals:** For each eigenvector field $i \in \{0, 1\}$, compute the binormal vector $b_i$ using the cross product with the image plane normal:
    $$ b_i = (0, 0, 1)^T \times v'_{\lambda_i} $$
    [S1, book]

2.  **Find Border Scaling Factors:** Determine the scaling factors $a_p$ and $a_n$ that define the tube width. This involves finding the set $B$ of scaling factors $s$ such that the response $R(s b_1)$ falls below a threshold $\varepsilon$:
    $$ B = \{s \mid R(s b_1) < \varepsilon \wedge s \in \mathbb{R}\} $$
    [S1, book]
    Then, identify the positive and negative bounds:
    $$ a_p = \min\{s \mid s \in B \wedge s \geq 0\} $$
    $$ a_n = \max\{s \mid s \in B \wedge s < 0\} $$
    [S1, book]
    *Note: The text implies $a_p$ and $a_n$ are derived from the same binormal direction (specifically $b_1$ is mentioned in the set definition, though the width uses both fields conceptually).* [S1, book]

3.  **Calculate Tube Width:** The width of the tube at the current point is defined by the sum of the scaling factors:
    $$ \text{width} = a_p + a_n $$
    [S1, book]

4.  **Calculate Position Ratio:** Determine the position ratio $p$ of the current point on the tube cross-section:
    $$ p = \frac{a_p}{a_p + a_n} \in [0, 1] $$
    [S1, book]

5.  **Calculate Tube Ratio:** Compute the ratio $r$ based on $p$ to describe the round surface of the tube. The value $p$ is squared in the logic to derive $r$:
    $$ r = \begin{cases} (1-2p)^2 & \text{if } p \geq 0 \\ -(1-2p)^2 & \text{if } p < 0 \end{cases} $$
    [S1, book]
    The resulting $r$ is in the range $[-1, 1]$ [S1, book].

6.  **Calculate New Normal:** Interpolate between the image plane normal and the first binormal $b_0$ using $r$:
    $$ n = (1-r)(0,0,1)^T + r b_0 $$
    [S1, book]

7.  **Shading:** Use the calculated normal $n$ for Phong shading on the surface [S1, book].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fabric_tube_rendering(image_response, v0, v1, epsilon, max_steps=100, step_size=0.01):
    """
    Renders fabric-like streamtubes on the surface using image-space analysis.
    
    Args:
        image_response: Function R(x,y) representing the composited image.
        v0: Eigenvector v'_lambda_0 at current pixel.
        v1: Eigenvector v'_lambda_1 at current pixel.
        epsilon: Threshold for border detection.
        max_steps: Maximum search iterations for border finding.
        step_size: Discrete step size for the search.
        
    Returns:
        Shading normal n as a 3-tuple.
    """
    # Image plane normal
    n_plane = (0.0, 0.0, 1.0)

    # Step 1: Calculate binormals b_i = n_plane x v'_lambda_i
    def cross(a, b):
        return (a[1]*b[2] - a[2]*b[1], a[2]*b[0] - a[0]*b[2], a[0]*b[1] - a[1]*b[0])

    b0 = cross(n_plane, v0)
    b1 = cross(n_plane, v1)

    # Step 2: Find border scaling factors a_p and a_n along b1
    # The source does not specify the exact search algorithm [G].
    # We perform a discrete search along the binormal direction.
    a_p = None
    a_n = None
    
    # Search for positive direction
    for k in range(1, max_steps + 1):
        s_pos = k * step_size
        if image_response(s_pos * b1[0], s_pos * b1[1]) < epsilon:
            a_p = s_pos
            break
            
    # Search for negative direction
    for k in range(1, max_steps + 1):
        s_neg = -k * step_size
        if image_response(s_neg * b1[0], s_neg * b1[1]) < epsilon:
            a_n = s_neg
            break

    # Edge case: fallback if borders are not found
    if a_p is None or a_n is None:
        return n_plane

    # Step 3 & 4: Position ratio p
    p = a_p / (a_p + a_n)

    # Step 5: Tube ratio r
    if p >= 0:
        r = (1.0 - 2.0 * p) ** 2
    else:
        r = -((1.0 - 2.0 * p) ** 2)

    # Step 6: Calculate new normal n = (1-r)*n_plane + r*b0
    n = (
        (1.0 - r) * n_plane[0] + r * b0[0],
        (1.0 - r) * n_plane[1] + r * b0[1],
        (1.0 - r) * n_plane[2] + r * b0[2]
    )
    return n
```

**Parameters:**
- $\varepsilon$: Threshold for border detection. Typical value not given in the sources. [S1, book]
- $v'_{\lambda_i}$: Eigenvectors in image space. [S1, book]

**Edge Cases and Pitfalls:**
- **Discontinuities:** Artifacts result from the local approach; discontinuities along a tube may occur because integration along the eigenvector field is not performed [S1, book].
- **Blurry Input:** Artifacts are caused by blurry input fields where borders cannot be found clearly. These are less visible if the fabric structure frequency is high [S1, book].
- **Undefined Borders:** If $a_p$ or $a_n$ cannot be found (e.g., response never drops below $\varepsilon$), the method fails to define a tube. The pseudocode includes a fallback to the plane normal [S1, book].

## Operating characteristics
- **Time Complexity:** Depends on the search for $a_p$ and $a_n$. Since it is a local approach without integration, it is likely $O(1)$ per pixel if the search range is bounded, or $O(k)$ where $k$ is the number of samples along the binormal [G].
- **Memory Complexity:** $O(1)$ per pixel, as it only requires local eigenvectors and image response values [G].
- **Strengths:** Provides a better spatial impression of fabric-like patterns without the computational cost of generating and rendering actual 3D tube geometry [S1, book].
- **Limitations:** Local approach leads to discontinuities. Sensitive to blurry input fields [S1, book].
- **When to use:** When visualizing tensor field data on arbitrary surfaces in image space and a fabric-like appearance is desired [S1, book].
- **When not to use:** When global continuity of streamlines is required or when the input field is very blurry [S1, book].

## Background
Fabric Tube Rendering was introduced in the context of fabric-like visualization of tensor field data on arbitrary surfaces in image space [S1, book]. It builds on the idea of interpreting the structure on the surface as streamtubes without actually creating tubes, similar to approaches in [18, 20] (cited in [S1, book]) [S1, book]. It is used to achieve a better spatial impression of fabric-like patterns, often in conjunction with bump mapping and postprocessing filters [S1, book].

## Sources
[S1] Fabric-Like Visualization of Tensor Field Data on Arbitrary Surfaces in Image Space, https://doi.org/10.1007/978-3-642-27343-8_4, fetched 2026-10-27 (book)
