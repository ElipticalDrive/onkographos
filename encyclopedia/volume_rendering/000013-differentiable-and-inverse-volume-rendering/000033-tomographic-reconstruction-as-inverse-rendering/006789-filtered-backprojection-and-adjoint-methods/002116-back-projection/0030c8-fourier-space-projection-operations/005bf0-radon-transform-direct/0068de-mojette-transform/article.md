# Mojette Transform

Also called: Discrete Radon Transform, Exact Discrete Radon Transform, Mojette Projection, Mojette Transformation

## Overview
The Mojette transform is an exact discrete version of the Radon transform that allows the projection of discrete data onto a discrete hyperplane [S1, unofficial]. Unlike the continuous Radon transform, which requires interpolation for arbitrary angles, the Mojette transform restricts projection directions to discrete angles defined by integer vectors, ensuring that line integrals pass exactly through pixel centers [S1, unofficial]. This induces regular sampling on the projections and eliminates interpolation errors for a finite number of angles [S1, unofficial]. It builds on [Radon Transform (Direct)](17030-radon-transform-direct.md) by replacing the continuous integral with a discrete sum constrained by integer geometry.

## How it works
**Inputs and Outputs:**
- Input: A discrete 2D image $f(k, l)$ defined on a Cartesian grid, where $k, l$ are integer coordinates.
- Output: A projection bin value $proj(t, \theta)$ for a specific discrete angle defined by integers $(p, q)$.

**Data Structures:**
- Image $f$: A 2D array or map indexed by integer coordinates $(k, l)$.
- Projection direction: Defined by two integers $(p, q)$, where $p$ is the displacement in pixels along the x-axis ($\vec{x}$) and $q$ is the displacement in pixels along the y-axis ($\vec{y}$) [S1, unofficial].
- Projection coordinate $t$: The position along the projection axis, orthogonal to the direction $(p, q)$.

**Mathematical Formulation:**
The continuous object $f(x, y)$ is modeled using Dirac pixels [S1, unofficial]:
$$ f(x, y) = \sum_{k=-\infty}^{+\infty} \sum_{l=-\infty}^{+\infty} f(k, l) \delta(x - \Delta_p k) \delta(y - \Delta_p l) $$
where $\delta(x) = 1$ if $x = 0$ and $0$ otherwise, and $\Delta_p$ is the pixel size [S1, unofficial].

The Radon transform at angle $\theta$ is:
$$ proj(t, \theta) = \int_{-\infty}^{+\infty} \int_{-\infty}^{+\infty} f(x, y) \delta(t + x \sin \theta - y \cos \theta) dx dy $$
Substituting the Dirac model yields:
$$ proj_\delta(t, \theta) = \sum_{k=-\infty}^{+\infty} \sum_{l=-\infty}^{+\infty} f(k, l) \delta(t + \Delta_p k \sin \theta - \Delta_p l \cos \theta) $$
For the projection to be non-zero, the argument of the delta function must be zero:
$$ t + \Delta_p k \sin \theta - \Delta_p l \cos \theta = 0 $$
Regular sampling is achieved only when $\tan \theta = q/p$ for integers $p, q$ [S1, unofficial]. In this case:
$$ \cos \theta = \frac{p}{\sqrt{p^2 + q^2}}, \quad \sin \theta = \frac{q}{\sqrt{p^2 + q^2}} $$
The condition for a pixel $(k, l)$ to contribute to bin $t$ becomes:
$$ t = \Delta_p (l \cos \theta - k \sin \theta) = \frac{\Delta_p}{\sqrt{p^2 + q^2}} (l p - k q) $$
Let $u = l p - k q$. The projection bin index $t$ is proportional to $u$. The value of the projection bin corresponding to $u$ is the sum of all $f(k, l)$ such that $l p - k q = u$ [S1, unofficial].

**Algorithm Steps:**
1. Define the discrete projection direction by integers $(p, q)$.
2. Initialize a projection array (sinogram row) with zeros. The range of indices depends on the image bounds and $(p, q)$.
3. Iterate over all pixels $(k, l)$ in the input image.
4. Calculate the projection coordinate value $u = l \cdot p - k \cdot q$.
5. Map $u$ to the appropriate bin index in the projection array.
6. Add $f(k, l)$ to the value at that bin index.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mojette_transform(image, p, q):
    """
    Computes the Mojette projection for direction (p, q).
    
    Args:
        image: 2D array f(k, l) where k is row index (y/l) and l is col index (x/k)? 
               Note: Standard matrix notation usually f(row, col). 
               Let's align with source: f(k, l) where k is x-index, l is y-index?
               Source says: "f(k, l) is the object value at integer coordinates (k, l)".
               Usually x=k, y=l.
               Projection coordinate u = l*p - k*q.
        
        p: Displacement in pixels along x-axis.
        q: Displacement in pixels along y-axis.
        
    Returns:
        projection: Array of summed values indexed by u.
    """
    # Determine image dimensions
    # Assuming image is accessed as image[y][x] or image[l][k]
    # Let's assume standard image coordinates: x is column, y is row.
    # Source: k is x, l is y.
    # So image[l][k] accesses pixel at x=k, y=l.
    
    height = image.shape[0] # Number of l values (y)
    width = image.shape[1]  # Number of k values (x)
    
    # Calculate range of u = l*p - k*q
    # Min u: min(l*p) - max(k*q)
    # Max u: max(l*p) - min(k*q)
    
    min_l, max_l = 0, height - 1
    min_k, max_k = 0, width - 1
    
    # Handle signs of p and q for min/max calculation
    if p >= 0:
        min_lp = min_l * p
        max_lp = max_l * p
    else:
        min_lp = max_l * p
        max_lp = min_l * p
        
    if q >= 0:
        min_kq = min_k * q
        max_kq = max_k * q
    else:
        min_kq = max_k * q
        max_kq = min_k * q
        
    min_u = min_lp - max_kq
    max_u = max_lp - min_kq
    
    # Initialize projection array
    # Size is max_u - min_u + 1
    size = max_u - min_u + 1
    projection = [0.0] * size
    
    # Offset to map u to array index 0..size-1
    offset = -min_u
    
    # Iterate over all pixels
    for l in range(height):
        for k in range(width):
            # Calculate projection coordinate
            u = l * p - k * q
            
            # Map to index
            idx = u + offset
            
            # Add to projection
            # Note: Source does not specify normalization by sqrt(p^2+q^2) 
            # for the *sum* itself, but the geometric position t is scaled.
            # The Mojette transform is often defined as the sum.
            # The source equation 4.4 implies the sum of f(k,l) for valid pixels.
            projection[idx] += image[l][k]
            
    return projection
```

**Parameters:**
- $(p, q)$: Integer vector defining the projection direction. Typical values are coprime integers to avoid redundant sampling. Not given in the sources.
- $\Delta_p$: Pixel size. Not given in the sources (often assumed 1).

**Edge Cases and Pitfalls:**
- **Degenerate Directions:** If $p=0$ and $q=0$, the direction is undefined.
- **Boundary Conditions:** The projection array size depends on the image bounds and $(p, q)$. Pixels near the boundary contribute to bins that might be outside the central range if not handled correctly.
- **Precision:** Since the transform is exact and discrete, floating-point precision issues are minimal compared to interpolated Radon transforms, provided integer arithmetic is used for $u$ [S1, unofficial].

## Subtopics

### Matus and Flusser Discrete Radon Transform
Defined on a $p \times p$ table where $p$ is a prime number [S1, unofficial]. The continuous Radon transform integral is replaced by a discrete sum on lines $k = ml + t \mod p$ for all intersection points $0 \le t \le p$, for a line slope $m$ such that $0 \le m \le p$ [S1, unofficial]. The image space is considered toroidal, meaning projection lines "wrap around" the image [S1, unofficial]. For an image $f(k, l)$, the transform is:
$$ R(t, m) = \begin{cases} \sum_{i=0}^{p-1} f(i, t) & \text{if } m=0 \\ \sum_{j=0}^{p-1} f((mj + t)\mod p, j) & \text{if } 0 < m \le p \end{cases} $$
This variant is chosen when a finite, cyclic geometry is required, such as in certain algebraic reconstruction techniques or discrete geometry applications [S1, unofficial]. It differs from the standard Mojette transform by enforcing toroidal topology and prime-sized dimensions [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ per projection direction, where $N$ is the number of pixels in the image, as each pixel is visited once [S1, unofficial].
- **Memory Complexity:** $O(W_{proj})$ for the projection array, where $W_{proj}$ is the width of the projection bin array, proportional to the image dimensions [S1, unofficial].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Linear with image size. The number of bins scales with the image dimensions and the slope $(p, q)$ [S1, unofficial].
- **Strengths:** Exact reconstruction for a finite set of angles; no interpolation errors; computationally efficient [S1, unofficial].
- **Limitations:** Limited to discrete angles defined by integer vectors $(p, q)$; cannot represent arbitrary continuous angles without approximation [S1, unofficial].
- **When to use:** Applications in discrete geometry, network coding, and medical image protection where exact discrete projections are required [S1, unofficial].
- **When not:** When arbitrary angle coverage is needed without discrete constraints.

## Background
The Mojette transform was developed in 1995 by Jean-Pierre Guédon in the IVC (Image and Video Communication) team at the IRCCyN laboratory [S1, unofficial]. It was introduced as an exact discrete version of the Radon transform to facilitate tomographic reconstruction and other applications requiring discrete projections [S1, unofficial]. Applications include discrete geometry, networks, and medical image protection [S1, unofficial].

## Sources
[S1] Reconstruction Tomographique Mojette, https://theses.hal.science/tel-00426920v1, fetched 2026-10-26 (unofficial)
