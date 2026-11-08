# Fabric Tube Rendering

Also called: Fabric-like visualization of tensor field data on arbitrary surfaces in image space, streamtube interpretation on surface

## Overview
Fabric Tube Rendering is a post-processing technique that interprets a composited image of a tensor field as streamtubes on the surface without creating actual 3D geometry. It uses binormals derived from eigenvectors and image-plane normals to determine tube position and width, creating a tube-like effect with proper spatial impression [S1, book]. It sets itself apart from [Stream Tube Generation](2707-stream-tube-generation.md) by avoiding explicit geometric construction and integration along the field, instead relying on local image-space analysis to compute shading normals.

## How it works
**Inputs and Outputs:**
- **Input:** A composited image $R(x,y)$ (or $G(x,y)$) representing the fabric structure, the eigenvectors $v'_{\lambda_0}, v'_{\lambda_1}$ in image space, and a threshold $\varepsilon$.
- **Output:** A shaded image with a fabric-like spatial impression using Phong shading.

**Data Structures:**
- Image pixels at coordinates $(x,y)$.
- Eigenvectors $v'_{\lambda_i}$ for $i \in \{0, 1\}$ at each pixel.
- Image plane normal $n_{plane} = (0,0,1)^T$.

**Steps:**

1.  **Calculate Binormals:** For each eigenvector field $i \in \{0, 1\}$, compute the binormal vector $b_i$ using the cross product with the image plane normal:
    $$ b_i = (0, 0, 1)^T \times v'_{\lambda_i} $$
    [S1, book]

2.  **Find Border Scaling Factors:** Determine the scaling factors $a_p$ and $a_n$ that define the tube width. This involves finding the set $B$ of scaling factors $s$ such that the response $R(s b_1)$ falls below a threshold $\varepsilon$:
    $$ B = \{s \mid R(s b_1) < \varepsilon \wedge s \in \mathbb{R}\} $$
    Then, identify the positive and negative bounds:
    $$ a_p = \min\{s \mid s \in B \wedge s \geq 0\} $$
    $$ a_n = \max\{s \mid s \in B \wedge s < 0\} $$
    [S1, book]
    *Note: The text implies $a_p$ and $a_n$ are derived from the same binormal direction (specifically $b_1$ is mentioned in the set definition, though the width uses both fields conceptually).*

3.  **Calculate Tube Width:** The width of the tube at the current point is defined by the sum of the scaling factors:
    $$ \text{width} = a_p + a_n $$
    [S1, book]

4.  **Calculate Position Ratio:** Determine the position ratio $p$ of the current point on the tube cross-section:
    $$ p = \frac{a_p}{a_p + a_n} \in [0, 1] $$
    [S1, book]

5.  **Calculate Tube Ratio:** Compute the ratio $r$ based on $p$ to describe the round surface of the tube. The value $p$ is squared in the logic to derive $r$:
    $$ r = \begin{cases} (1-2p)^2 & \text{if } p \geq 0 \\ -(1-2p)^2 & \text{if } p < 0 \end{cases} $$
    The resulting $r$ is in the range $[-1, 1]$.
    [S1, book]

6.  **Calculate New Normal:** Interpolate between the image plane normal and the first binormal $b_0$ using $r$:
    $$ n = (1-r)(0,0,1)^T + r b_0 $$
    [S1, book]

7.  **Shading:** Use the calculated normal $n$ for Phong shading on the surface.
    [S1, book]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fabric_tube_rendering(image_response, eigenvector_0, eigenvector_1, epsilon):
    """
    Renders fabric-like streamtubes on the surface using image-space analysis.
    
    Args:
        image_response: Function or array R(x,y) representing the composited image.
        eigenvector_0: Eigenvector v'_lambda_0 at current pixel.
        eigenvector_1: Eigenvector v'_lambda_1 at current pixel.
        epsilon: Threshold for border detection.
        
    Returns:
        Shading normal n.
    """
    # Image plane normal
    n_plane = Vector(0, 0, 1)
    
    # Step 1: Calculate binormals
    # b_i = n_plane x v'_lambda_i
    b_0 = cross_product(n_plane, eigenvector_0)
    b_1 = cross_product(n_plane, eigenvector_1)
    
    # Step 2: Find border scaling factors
    # Find set B = {s | R(s * b_1) < epsilon}
    # In practice, this requires searching along the binormal direction b_1
    # from the current pixel position until the response drops below epsilon.
    # We need the smallest positive s (a_p) and largest negative s (a_n).
    
    # Search for positive direction
    a_p = find_min_positive_scaling(image_response, b_1, epsilon)
    
    # Search for negative direction
    a_n = find_max_negative_scaling(image_response, b_1, epsilon)
    
    # Handle edge cases where borders are not found
    if a_p is None or a_n is None:
        # Fallback to plane normal if tube structure is not detected
        return n_plane
    
    # Step 3: Tube width (not directly used for shading, but defines p)
    # width = a_p + a_n
    
    # Step 4: Position ratio
    p = a_p / (a_p + a_n)
    
    # Step 5: Tube ratio r
    # r describes the ratio between normal on plane and normal pointing to camera
    if p >= 0:
        r = (1 - 2 * p) ** 2
    else:
        r = -((1 - 2 * p) ** 2)
        
    # Step 6: Calculate new normal
    # n = (1-r)*n_plane + r*b_0
    n = scale_vector(n_plane, 1 - r) + scale_vector(b_0, r)
    
    # Normalize if necessary (though interpolation of unit vectors may not be unit)
    n = normalize(n)
    
    return n

def find_min_positive_scaling(response_func, direction, epsilon):
    """
    Finds the smallest positive scaling factor s such that response(s*direction) < epsilon.
    This is a local search along the direction vector.
    """
    # Implementation detail: Ray march or sample along direction
    # Since the source does not specify the exact search algorithm, 
    # we assume a discrete search or root finding.
    # [G] Placeholder for specific search implementation as source does not detail the solver.
    # Typically, one would sample s = 0, delta, 2*delta... until condition met.
    # For the purpose of this article, we assume a function exists that returns this value.
    return 0.0 # Placeholder: Not given in the sources.

def find_max_negative_scaling(response_func, direction, epsilon):
    """
    Finds the largest negative scaling factor s such that response(s*direction) < epsilon.
    """
    # Similar to above, search in negative direction.
    return 0.0 # Placeholder: Not given in the sources.
```

**Parameters:**
- $\varepsilon$: Threshold for border detection. Typical value not given in the sources.
- $v'_{\lambda_i}$: Eigenvectors in image space.

**Edge Cases and Pitfalls:**
- **Discontinuities:** Artifacts result from the local approach; discontinuities along a tube may occur because integration along the eigenvector field is not performed [S1, book].
- **Blurry Input:** Artifacts are caused by blurry input fields where borders cannot be found clearly. These are less visible if the fabric structure frequency is high [S1, book].
- **Undefined Borders:** If $a_p$ or $a_n$ cannot be found (e.g., response never drops below $\varepsilon$), the method fails to define a tube. The pseudocode includes a fallback to the plane normal.

## Operating characteristics
- **Time Complexity:** Depends on the search for $a_p$ and $a_n$. Since it is a local approach without integration, it is likely $O(1)$ per pixel if the search range is bounded, or $O(k)$ where $k$ is the number of samples along the binormal.
- **Memory Complexity:** $O(1)$ per pixel, as it only requires local eigenvectors and image response values.
- **Strengths:** Provides a better spatial impression of fabric-like patterns without the computational cost of generating and rendering actual 3D tube geometry. It works directly in image space.
- **Limitations:** Local approach leads to discontinuities. Sensitive to blurry input fields.
- **When to use:** When visualizing tensor field data on arbitrary surfaces in image space and a fabric-like appearance is desired.
- **When not to use:** When global continuity of streamlines is required or when the input field is very blurry.

## Background
Fabric Tube Rendering was introduced in the context of fabric-like visualization of tensor field data on arbitrary surfaces in image space. It builds on the idea of interpreting the structure on the surface as streamtubes without actually creating tubes, similar to approaches in [18, 20] (cited in [S1, book]). It is used to achieve a better spatial impression of fabric-like patterns, often in conjunction with bump mapping and postprocessing filters.

## Sources
[S1] Fabric-Like Visualization of Tensor Field Data on Arbitrary Surfaces in Image Space, https://doi.org/10.1007/978-3-642-27343-8_4, fetched 2026-10-27 (book)
