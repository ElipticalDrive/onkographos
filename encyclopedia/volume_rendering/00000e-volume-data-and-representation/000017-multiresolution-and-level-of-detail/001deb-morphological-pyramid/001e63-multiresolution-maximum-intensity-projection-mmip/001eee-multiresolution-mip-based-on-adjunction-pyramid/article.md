# Multiresolution MIP based on adjunction pyramid
Also called: Progressive refinement from adjunction pyramid

## Overview
This method performs progressive Maximum Intensity Projection (MIP) volume rendering by refining the image directly in the 2D plane using an adjunction pyramid, avoiding 3D volume reconstruction [S1, unofficial]. It leverages the commutativity of MIP and dilation to update the projection by taking the maximum of the expanded detail projection and the previous approximation [S1, unofficial]. Unlike general detail pyramids that require 3D reconstruction, this variant exploits the specific properties of adjunction pyramids where detail signals are non-negative, allowing for simpler, purely 2D progressive refinement [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** An adjunction detail pyramid sequence $d_0, d_1, \dots, d_{L-1}$ and the coarsest approximation $f_L$ of the input volume data $f$. The parameter $\Theta$ defines the viewing coordinate system orientation [S1, unofficial].
*   **Output:** A sequence of progressively refined approximation images $M_\Theta^{(j)}(f)$ for $j=L, \dots, 0$ [S1, unofficial].

**Data Structures:**
*   **Pyramid Levels:** The pyramid has depth $L$. Each level $j$ contains detail data $d_{j-1}$ (for $j=1 \dots L$) and the coarsest approximation $f_L$.
*   **Operators:**
    *   $\psi_A$: The 3D analysis operator (REDUCE) of the adjunction pyramid.
    *   $\tilde{\psi}_A$: The 2D synthesis operator (EXPAND) applied in the image plane. Specifically, $\tilde{\psi}_A^{k}$ denotes $k$-fold composition of $\tilde{\psi}_A$ with itself [S1, unofficial].
    *   $\vee$: The pointwise maximum operator, defined as $(g_0 \vee g_1)(x) = \max(g_0(x), g_1(x))$ [S1, unofficial].

**Algorithm Steps:**
1.  **Initialize:** Choose the viewing orientation $\Theta$. Compute the coarsest approximation image by applying the 2D synthesis operator $\tilde{\psi}_A$ $L$ times to the MIP of the coarsest volume approximation $f_L$:
    $$ M_\Theta^{(L)}(f) = \tilde{\psi}_A^L (M_\Theta(f_L)) $$
    [S1, unofficial]

2.  **Progressive Refinement:** Loop from $j = L$ down to $1$:
    a.  Compute the MIP of the detail signal at level $j-1$: $P = M_\Theta(d_{j-1})$.
    b.  Expand this projection $j-1$ times using the 2D synthesis operator: $E = \tilde{\psi}_A^{j-1}(P)$.
    c.  Update the current approximation by taking the maximum of the expanded detail projection and the previous approximation:
        $$ M_\Theta^{(j-1)}(f) = E \vee M_\Theta^{(j)}(f) $$
        [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multiresolution_mip_adjunction_pyramid(detail_pyramid, coarsest_approx, L, theta):
    """
    detail_pyramid: list of 3D volumes [d_0, d_1, ..., d_{L-1}]
    coarsest_approx: 3D volume f_L
    L: integer, decomposition depth
    theta: viewing orientation parameter
    """
    # Step 1: Coarsest approximation
    # M_Theta(f_L) computes the MIP of the coarsest volume along theta
    current_mip = mip_2d(coarsest_approx, theta)
    
    # Apply synthesis operator L times to expand to full resolution
    for _ in range(L):
        current_mip = expand_2d(current_mip)
        
    # Store the coarsest image if needed for output sequence
    # images[L] = current_mip

    # Step 2: Progressive refinement from L down to 1
    for j in range(L, 0, -1):
        # d_{j-1} is the detail at index j-1 in the list
        detail_vol = detail_pyramid[j-1]
        
        # Compute MIP of the detail signal
        detail_mip = mip_2d(detail_vol, theta)
        
        # Expand the detail projection j-1 times
        expanded_detail = detail_mip
        for _ in range(j-1):
            expanded_detail = expand_2d(expanded_detail)
            
        # Take the maximum with the previous approximation
        # M_Theta^{(j-1)}(f) = expanded_detail ∨ M_Theta^{(j)}(f)
        current_mip = pointwise_max(current_mip, expanded_detail)
        
        # Store current_mip as the approximation at level j-1
        # images[j-1] = current_mip

    return images
```

**Parameters:**
*   $L$: Decomposition depth. Typical values depend on the resolution of the volume data [S1, unofficial].
*   $\Theta$: Viewing orientation. Determines the projection direction [S1, unofficial].

**Pitfalls:**
*   **Monotonicity:** Because detail signals $d_{j-1}$ in an adjunction pyramid are non-negative, the projections $M_\Theta^{(j-1)}(f)$ are guaranteed to be greater than or equal to $M_\Theta^{(j)}(f)$. The image intensity increases pointwise as one goes down the pyramid (from coarse to fine) [S1, unofficial].
*   **Precision:** Standard floating-point precision is sufficient, but care should be taken with the expansion operator to avoid aliasing artifacts if the underlying morphological filter is not properly designed [S1, unofficial].

## Subtopics

### Multiresolution MIP based on approximation pyramid
A variant that uses an approximation pyramid instead of a detail pyramid. It avoids 3D reconstruction by applying MIP directly to approximation data at each level, which is suitable when 3D dilation is inefficient on compressed data [S1, unofficial].
*   **Difference:** Uses approximation data $f_0 \dots f_L$ rather than detail data. The synthesis operator must be a dilation [S1, unofficial].
*   **Algorithm:** Loop $j$ from $L$ down to $0$. Compute $M_\Theta^{(j)}(f) = \tilde{\psi}_A^j (M_\Theta(f_j))$ [S1, unofficial].
*   **Memory:** Requires storing all approximation data $f_0 \dots f_L$, which uses memory storage of the same order of magnitude as storing detail and coarsest approximation data [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The refinement is performed in 2D. For each level, MIP computation is $O(N)$ where $N$ is the number of voxels in the detail volume, and expansion is $O(M)$ where $M$ is the number of pixels in the image. The total time is dominated by the MIP computations at each level.
*   **Memory Complexity:** Requires storing the pyramid data. For the adjunction pyramid, this includes detail volumes and the coarsest approximation. The approximation pyramid variant requires storing all approximation levels, which is memory-intensive [S1, unofficial].
*   **Strengths:** Direct 2D refinement avoids expensive 3D reconstruction steps required by general detail pyramids. It is efficient for progressive rendering where early coarse levels provide a quick preview [S1, unofficial].
*   **Limitations:** Relies on the specific properties of adjunction pyramids (non-negative details). Not directly applicable to other pyramid types without modification [S1, unofficial].

## Background
This method is part of the family of multiresolution MIP techniques using morphological pyramids. It was introduced in the context of comparing different morphological pyramid structures for volume rendering [S1, unofficial]. The approach builds on the commutativity of MIP and dilation, allowing the projection to be refined in the image plane [S1, unofficial]. It is similar to wavelet splatting but replaces linear summation with maximum computation and linear wavelet filters with morphological filters [S1, unofficial].

## Sources
[S1] Comparison of Morphological Pyramids for Multiresolution MIP Volume Rendering, https://doi.org/10.2312/vissym/vissym02/061-070, fetched 2026-10-29 (unofficial)
