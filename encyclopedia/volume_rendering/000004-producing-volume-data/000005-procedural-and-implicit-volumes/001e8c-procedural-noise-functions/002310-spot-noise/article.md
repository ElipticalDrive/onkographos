# Spot Noise

Also called: Impulse convolution noise, Sparse convolution noise

## Overview
Spot Noise is a procedural volume rendering technique that synthesizes structured textures by convolving a sparse, uniform distribution of impulses with an arbitrary spatially defined kernel [S1, unofficial]. It solves the problem of generating coherent, non-periodic surface-like details in volumetric data without the grid artifacts common in value-based noise [S1, unofficial]. Unlike its sibling [Worley Noise](1726-worley-noise.md), which relies on distance to the nearest seed point, Spot Noise uses a continuous kernel convolution, allowing for smoother, more organic patterns and flexible kernel shapes [S1, unofficial].

## How it works
**Inputs:** Query point $P = (x, y, z)$; impulse generation parameters (density, jitter seed); kernel definition $K(\mathbf{u})$.
**Outputs:** Scalar density value $V(P)$.

**Data Structures:**
*   **Impulse List:** A sparse set of points $\{I_j\}$ with associated random values or weights. In real-time implementations, these are generated on-the-fly using a hash function to map grid cells to impulse positions [S1, unofficial].
*   **Kernel:** A function $K(\mathbf{u})$ centered at an impulse, typically an elliptical Gaussian or Gabor filter [S1, unofficial].

**Steps:**
1.  **Impulse Generation:** For a query point $P$, determine the relevant neighborhood of impulses. In a uniform distribution, this is often done by hashing the integer coordinates of the cell containing $P$ to generate a pseudo-random offset, creating a jittered impulse position $I_j$ within that cell [S1, unofficial].
2.  **Distance Calculation:** Compute the vector $\mathbf{u} = P - I_j$ from the impulse to the query point [S1, unofficial].
3.  **Kernel Evaluation:** Evaluate the kernel function $K(\mathbf{u})$ at the offset $\mathbf{u}$. For an elliptical Gaussian, this involves scaling $\mathbf{u}$ by the kernel's orientation and scale parameters [S1, unofficial].
4.  **Summation:** Sum the contributions of all relevant impulses in the neighborhood:
    $$ V(P) = \sum_{j} w_j \cdot K(P - I_j) $$
    where $w_j$ is the weight of impulse $j$ [S1, unofficial].

**Parameters:**
*   **Impulse Density:** Controls the average number of impulses per unit volume. Typical values depend on the desired feature size [S1, unofficial].
*   **Kernel Width:** Controls the spread of the influence of each impulse [S1, unofficial].
*   **Jitter Seed:** Ensures reproducibility of the impulse distribution [S1, unofficial].

**Edge Cases:**
*   **Boundary Conditions:** Impulses near the domain boundary may have incomplete neighborhoods, leading to edge artifacts. [G] A common fix is to wrap coordinates or extend the impulse generation range.
*   **Precision:** High impulse density can lead to numerical instability if not handled with care. [G] Use double precision for kernel evaluation if necessary.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def spot_noise(p, impulse_density, kernel_func, seed):
    """
    Compute spot noise value at point p.
    """
    # Determine grid cell containing p
    cell = floor(p)
    
    # Generate pseudo-random impulse position in the cell
    # Using a hash function to get a deterministic jitter
    hash_val = hash_function(cell + seed)
    jitter = vector(hash_val % 1.0, (hash_val * 1.234) % 1.0, (hash_val * 5.678) % 1.0)
    impulse_pos = cell + jitter
    
    # Calculate distance vector
    u = p - impulse_pos
    
    # Evaluate kernel
    # Note: In a full implementation, multiple impulses per cell or neighboring cells might be considered
    # For simplicity, we assume one impulse per cell for this example
    contribution = kernel_func(u)
    
    return contribution
```

## Subtopics

### [Locally Controlled Spot Noise](4599-locally-controlled-spot-noise.md)
Extends standard Spot Noise by allowing spatial control over impulse density and kernel orientation/scale via underlying data fields, enabling the synthesis of fuzzy surface details.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per query if the number of contributing impulses is bounded and fixed. However, finding the relevant impulses can be $O(N)$ if a brute-force search is used, where $N$ is the number of impulses in the neighborhood [S1, unofficial].
*   **Memory Complexity:** $O(N)$ to store the impulse list, where $N$ is the total number of impulses. For on-the-fly generation, memory usage is minimal [S1, unofficial].
*   **Strengths:** Flexible kernel shapes; coherent, structured patterns; no grid artifacts [S1, unofficial].
*   **Limitations:** Can be computationally expensive if many impulses contribute; requires careful tuning of impulse density and kernel width [S1, unofficial].

## Background
Spot Noise was introduced as a sparse convolution noise method, building on earlier work in procedural texture synthesis [S1, unofficial]. It was developed to provide more control over the spatial distribution of features compared to traditional grid-based noise functions [S1, unofficial].

## Sources
[S1] Volumetric Spot Noise for Procedural 3D Shell Texture Synthesis, https://hal.science/hal-02413269v1, fetched 2026-10-26 (unofficial)
