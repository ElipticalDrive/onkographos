# Multiresolution MIP based on adjunction pyramid
Also called: Progressive refinement from adjunction pyramid

## Overview
This method performs progressive Maximum Intensity Projection (MIP) volume rendering by refining the image directly in the 2D plane using an adjunction pyramid, avoiding 3D volume reconstruction [S1]. It leverages the commutativity of MIP and dilation to update the projection by taking the maximum of the expanded detail projection and the previous approximation [S1]. Unlike general detail pyramids that require 3D reconstruction, this variant exploits the specific properties of adjunction pyramids where detail signals are non-negative, allowing for simpler, purely 2D progressive refinement [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** An adjunction detail pyramid sequence $d_0, d_1, \dots, d_{L-1}$ and the coarsest approximation $f_L$ of the input volume data $f$ [S1]. The parameter $\Theta$ defines the viewing coordinate system orientation [S1].
*   **Output:** A sequence of progressively refined approximation images $M_\Theta^{(j)}(f)$ for $j=L, \dots, 0$ [S1].

**Data Structures:**
*   **Pyramid Levels:** The pyramid has depth $L$ [S1]. Each level $j$ corresponds to detail data $d_{j-1}$ (for $j=1 \dots L$) and the coarsest approximation $f_L$ [S1].
*   **Operators:**
    *   $\psi_A$: The 3D analysis operator (REDUCE) of the adjunction pyramid [S1].
    *   $\tilde{\psi}_A$: The 2D synthesis operator (EXPAND) applied in the image plane, where $\tilde{\psi}_A^{k}$ denotes $k$-fold composition with itself [S1].
    *   $\vee$: The pointwise maximum operator, defined as $(g_0 \vee g_1)(x) = \max(g_0(x), g_1(x))$ [S1].

**Algorithm Steps:**
1.  **Initialize:** Choose the viewing orientation $\Theta$ [S1]. Compute the coarsest approximation image by applying the 2D synthesis operator $\tilde{\psi}_A$ $L$ times to the MIP of the coarsest volume approximation $f_L$:
    $$ M_\Theta^{(L)}(f) = \tilde{\psi}_A^L (M_\Theta(f_L)) $$
    [S1]

2.  **Progressive Refinement:** Loop from $j = L$ down to $1$ [S1].
    a.  Compute the MIP of the detail signal at level $j-1$: $P = M_\Theta(d_{j-1})$ [S1].
    b.  Expand this projection $j-1$ times using the 2D synthesis operator: $E = \tilde{\psi}_A^{j-1}(P)$ [S1].
    c.  Update the current approximation by taking the maximum of the expanded detail projection and the previous approximation:
        $$ M_\Theta^{(j-1)}(f) = E \vee M_\Theta^{(j)}(f) $$
        [S1]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multiresolution_mip_adjunction(detail_pyramid, coarsest_approx, L, theta):
    # detail_pyramid: list of 3D volumes [d_0, d_1, ..., d_{L-1}]
    # coarsest_approx: 3D volume f_L
    # L: integer, decomposition depth
    # theta: viewing orientation parameter (direction vector)
    
    # Standard operations (self-contained):
    # max_along_rays(volume, direction): projects 3D volume along direction by taking max along rays
    # dilation_2d(image): applies 2D dilation (synthesis operator) to image
    
    # Step 1: Coarsest approximation
    current_mip = max_along_rays(coarsest_approx, theta)
    for _ in range(L):
        current_mip = dilation_2d(current_mip)
        
    # Step 2: Progressive refinement
    for j in range(L, 0, -1):
        detail_proj = max_along_rays(detail_pyramid[j-1], theta)
        expanded_detail = detail_proj
        for _ in range(j-1):
            expanded_detail = dilation_2d(expanded_detail)
        current_mip = max(current_mip, expanded_detail)
        
    return current_mip
```

**Parameters:**
*   $L$: Decomposition depth [S1].
*   $\Theta$: Viewing orientation. Determines the projection direction [S1].

**Pitfalls:**
*   **Monotonicity:** Because detail signals $d_{j-1}$ in an adjunction pyramid are non-negative, the projections $M_\Theta^{(j-1)}(f)$ are guaranteed to be greater than or equal to $M_\Theta^{(j)}(f)$ [S1]. The image intensity increases pointwise as one goes down the pyramid (from coarse to fine) [S1].

## Subtopics

### Multiresolution MIP based on approximation pyramid
A variant that uses an approximation pyramid instead of a detail pyramid [S1]. It avoids 3D reconstruction by applying MIP directly to approximation data at each level, which is suitable when 3D dilation is inefficient on compressed data [S1]. It differs from the adjunction variant by operating on approximation sequences $f_0 \dots f_L$ rather than detail sequences, and requires the synthesis operator to be a dilation [S1]. It is chosen when memory allows storing all approximation levels, as it uses storage of the same order of magnitude as storing detail and coarsest approximation data [S1]. See [Multiresolution Maximum Intensity Projection (MMIP)](1745-multiresolution-maximum-intensity-projection-mmip.md).

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1].
*   **Memory Complexity:** Not given in the sources [S1].
*   **Strengths:** Direct 2D refinement avoids expensive 3D reconstruction steps required by general detail pyramids [S1].
*   **Limitations:** Relies on the specific properties of adjunction pyramids (non-negative details) [S1].

## Background
This method was introduced in the context of comparing different morphological pyramid structures for volume rendering [S1]. The approach builds on the commutativity of MIP and dilation, allowing the projection to be refined in the image plane [S1]. It is similar to wavelet splatting but replaces linear summation with maximum computation and linear wavelet filters with morphological filters [S1].

## Sources
[S1] Comparison of Morphological Pyramids for Multiresolution MIP Volume Rendering, https://doi.org/10.2312/vissym/vissym02/061-070, fetched 2026-10-29
