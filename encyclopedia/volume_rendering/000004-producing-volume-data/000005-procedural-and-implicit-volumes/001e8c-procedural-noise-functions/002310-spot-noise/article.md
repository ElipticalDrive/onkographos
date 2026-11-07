# Spot Noise

Also called: Impulse convolution noise, Sparse convolution noise

## Overview
Spot Noise is a procedural noise method that synthesizes structured textures by convolving a uniform distribution of impulses with an arbitrary spatially defined kernel [S2, unofficial]. It develops patterns after many thousands of tiny drops, representing a flow field or spatial distribution [S1, unofficial]. The method can also be implemented using a spectral approach to achieve the desired result [S1, unofficial].

## How it works
**Inputs:** Query point $\mathbf{P} = (x, y, z)$; impulse density parameter; kernel definition $K(\mathbf{u})$ [G].
**Outputs:** Scalar density value $V(\mathbf{P})$ [G].

**Data Structures:**
*   **Impulse List:** A sparse set of points $\{I_j\}$ representing the uniform distribution of impulses [S2, unofficial]. In on-the-fly implementations, these are generated using jittering and pseudorandom number generation [S2, unofficial].
*   **Kernel:** A function $K(\mathbf{u})$ centered at an impulse, spatially defined and arbitrary [S2, unofficial].

**Steps:**
1.  **Impulse Generation:** For a query point $\mathbf{P}$, determine the relevant neighborhood of impulses. On-the-fly generation uses jittering and pseudorandom number generation to place impulses within the spatial domain [S2, unofficial].
2.  **Kernel Evaluation:** Evaluate the kernel function $K(\mathbf{u})$ at the offset $\mathbf{u}$ from each impulse to the query point.
3.  **Summation:** Sum the kernel contributions from all relevant impulses in the neighborhood to compute the final value.

**Parameters:**
*   **Impulse Density:** Controls the average number of impulses per unit volume [G].
*   **Kernel Definition:** Defines the shape and spatial falloff of each impulse [S2, unofficial].

**Edge Cases:**
*   **Boundary Conditions:** Impulses near the domain boundary may have incomplete neighborhoods, leading to edge artifacts. [G] A common fix is to wrap coordinates or extend the impulse generation range.
*   **Precision:** High impulse density can lead to numerical instability if not handled with care. [G] Use double precision for kernel evaluation if necessary.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def spot_noise(p, impulse_density, kernel_func, seed):
    """
    Compute spot noise value at point p.
    Implements on-the-fly impulse generation using jittering and pseudorandom number generation [S2, unofficial].
    """
    # 1. Impulse Generation: Determine grid cell and generate jittered impulse position
    cell = tuple(int(c) for c in p)
    # Deterministic pseudorandom offset using standard hash
    hash_val = hash(cell + (seed,))
    jitter = (
        (hash_val % 1000) / 1000.0,
        ((hash_val * 7) % 1000) / 1000.0,
        ((hash_val * 13) % 1000) / 1000.0
    )
    impulse_pos = tuple(cell[i] + jitter[i] for i in range(3))
    
    # 2. Kernel Evaluation: Compute offset vector and evaluate kernel
    u = tuple(p[i] - impulse_pos[i] for i in range(3))
    contribution = kernel_func(u)
    
    # 3. Summation: Return contribution (full implementation sums over neighborhood)
    return contribution
```

## Subtopics

### [Locally Controlled Spot Noise](4599-locally-controlled-spot-noise.md)
Extends standard Spot Noise by allowing spatial control over impulse density and kernel orientation/scale via underlying data fields, enabling the synthesis of fuzzy surface details. Unlike standard Spot Noise which uses a uniform impulse distribution, LCSN rejects impulses based on a local density factor, making it suitable for authoring textures where feature density varies across the domain.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Flexible kernel shapes; coherent, structured patterns [S2, unofficial].
*   **Limitations:** Not given in the sources.

## Background
Spot Noise is a sparse convolution noise method where a uniform distribution of impulses is convolved with an arbitrary kernel spatially defined [S2, unofficial]. It develops patterns after many thousands of tiny drops, representing a flow field, and Van Wijk presented a spectral approach to achieving this as well [S1, unofficial].

## Sources
[S1] Flow Visualization Techniques for CDF using Volume Rendering, https://www.osti.gov/servlets/purl/790542, fetched 2026-10-26 (unofficial)
[S2] Volumetric Spot Noise for Procedural 3D Shell Texture Synthesis, https://hal.science/hal-02413269v1, fetched 2026-10-26 (unofficial)
