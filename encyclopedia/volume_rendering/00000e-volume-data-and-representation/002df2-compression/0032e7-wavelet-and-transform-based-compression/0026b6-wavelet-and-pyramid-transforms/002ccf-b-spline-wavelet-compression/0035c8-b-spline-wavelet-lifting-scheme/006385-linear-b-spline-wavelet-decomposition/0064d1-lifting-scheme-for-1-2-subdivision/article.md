# Lifting scheme for 1/2 subdivision

Also called: 1/2 subdivision lifting scheme

## Overview
This method applies the lifting scheme to linear B-spline wavelet decomposition specifically for 1/2 subdivision, handling the topological variations that arise during hierarchical volume data compression [S1, unofficial]. It distinguishes between hypervolume, volume, face, and edge cases based on the subdivision step, automatically managing boundary conditions without additional examination [S1, unofficial]. It builds on [Linear B-spline Wavelet Decomposition](18893-linear-b-spline-wavelet-decomposition.md).

## How it works
**Inputs and Outputs:**
*   **Input:** A grid of scalar values (e.g., entropy, density) at vertex locations. The grid may be 1D, 2D, 3D, or 4D (hypervolume).
*   **Output:** Compressed wavelet coefficients and approximation coefficients.

**Data Structures:**
The method operates on vertex values categorized by their topological role in the subdivision lattice:
*   $o$: Original neighbor vertices.
*   $a$: New vertices added at the center of edges (1D).
*   $A$: New vertices added at the center of faces (2D).
*   $\mathbf{A}$: New vertices added at the center of volumes (3D).
*   $\mathbf{A}$: New vertices added at the center of hypervolumes (4D).

**Parameters:**
*   $a, b$: Parameters derived from the interpolation assumptions (linear, bilinear, trilinear). The specific numerical values are not explicitly defined as constants like 0.5 in the source text but are derived from the interpolation factors $2a$, $4a^2$, and $8a^3$ [S1, unofficial].

**Steps:**
The encoding and decoding processes depend on the dimensionality of the case being processed. The source text describes a hierarchy starting from the highest dimension (hypervolume) down to 1D.

1.  **Hypervolume Case (4D):**
    *   **Assumptions:**
        *   Values at vertices $a$ are linear interpolations of neighbor $o$ vertices multiplied by $2a$ [S1, unofficial].
        *   Values at vertices $A$ are bilinear interpolations of neighbor $o$ vertices multiplied by $4a^2$ [S1, unofficial].
        *   Values at vertices $\mathbf{A}$ are trilinear interpolations of neighbor $o$ vertices multiplied by $8a^3$ [S1, unofficial].
        *   Values at vertices $A$ and $\mathbf{A}$ vanish (are zero) [S1, unofficial].
    *   **Encoding:**
        *   Apply $s$-lift encoding. The theoretical 4D $s$-lift mask reduces to $s$-liftencode$(a, b)$ due to vanishing values [S1, unofficial].
        *   Apply $w$-lift encoding. The mask reduces to $w$-liftencode$(a, b)$ [S1, unofficial].
    *   **Decoding:**
        *   Apply $s$-lift decoding. Rename factor $a$ to obtain $s$-liftdecode$(a, b)$ [S1, unofficial].
        *   Apply $w$-lift decoding. The final $w$-lift operation reduces to $w$-liftencode$(a, b)$ (inverse of encoding) [S1, unofficial].

2.  **Volume Case (3D):**
    *   **Assumptions:** Values at vertices $\mathbf{A}$ are trilinear interpolations of neighbor $o$ vertices [S1, unofficial]. If values at $\mathbf{A}$ are available, they must be computed *only* from $o$ vertices to maintain consistency [S1, unofficial].
    *   **Encoding:**
        *   Apply $W$-liftencode$(a, b)$ [S1, unofficial].
        *   Apply $S$-liftencode$(a, b)$ [S1, unofficial].
    *   **Decoding:**
        *   Apply $s$-liftdecode$(a, b)$ [S1, unofficial].
        *   Apply $w$-liftdecode$(a, b)$ [S1, unofficial].

3.  **Face Case (2D):**
    *   **Assumptions:** Values at vertices $A$ are bilinear interpolations of neighbor $o$ vertices [S1, unofficial]. If values at $A$ are available, compute them *only* from $o$ vertices [S1, unofficial].
    *   **Encoding:**
        *   Apply $W$-liftencode$(a, b)$ [S1, unofficial].
    *   **Decoding:**
        *   Not explicitly detailed for the face case alone, but implied to use the inverse of the encoding masks [S1, unofficial].

4.  **Edge Case (1D):**
    *   **Assumptions:** Values at vertices $a$ are linear interpolations of neighbor $o$ vertices [S1, unofficial]. If values at $a$ are available, compute them *only* from $o$ vertices [S1, unofficial].
    *   **Encoding/Decoding:**
        *   Apply masks (1) and (2) from Section 4 of the source [S1, unofficial].

**Pitfalls and Edge Cases:**
*   **Boundary Handling:** The volume, face, and edge cases automatically cover boundary volumes, boundary faces, and boundary edges of the domain. No additional boundary case examination is necessary [S1, unofficial].
*   **Interpolation Consistency:** When computing values for new vertices ($a, A, \mathbf{A}$), one must strictly use the neighbor $o$ vertices. Using already computed new vertices for further interpolation violates the assumptions of the lifting scheme [S1, unofficial].
*   **Mask Reduction:** The masks for $s$-lift and $w$-lift reduce significantly because values at certain vertex types ($A, \mathbf{A}$) vanish. Implementers must ensure the reduced masks are used rather than the full theoretical masks [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def lift_encode_hypervolume(data, a, b):
    """
    Encodes 4D hypervolume data using the reduced lifting scheme.
    Assumes values at a, A, A vanish or are interpolated from o.
    """
    # Step 1: s-lift encode
    # The mask reduces to s-liftencode(a, b)
    # Implementation depends on the specific reduced mask structure
    # which is described as having vanishing values at A and A.
    data = s_lift_encode_step(data, a, b)
    
    # Step 2: w-lift encode
    # The mask reduces to w-liftencode(a, b)
    data = w_lift_encode_step(data, a, b)
    
    return data

def lift_decode_hypervolume(data, a, b):
    """
    Decodes 4D hypervolume data.
    """
    # Step 1: w-lift decode
    # The final w-lift operation reduces to w-liftencode(a, b)
    # Note: The source states decoding is performed by applying
    # w-liftencode(a, b) which implies the inverse structure
    # or a specific reduced mask for decoding.
    data = w_lift_decode_step(data, a, b)
    
    # Step 2: s-lift decode
    # Rename factor a to obtain s-liftdecode(a, b)
    data = s_lift_decode_step(data, a, b)
    
    return data

def lift_encode_volume(data, a, b):
    """
    Encodes 3D volume data.
    Assumes values at A are trilinear interpolations of neighbor o vertices.
    """
    # Ensure values at A are computed only from o vertices
    # Apply W-liftencode(a, b)
    data = w_lift_encode_step(data, a, b)
    
    # Apply S-liftencode(a, b)
    data = s_lift_encode_step(data, a, b)
    
    return data

def lift_decode_volume(data, a, b):
    """
    Decodes 3D volume data.
    """
    # Apply s-liftdecode(a, b)
    data = s_lift_decode_step(data, a, b)
    
    # Apply w-liftdecode(a, b)
    data = w_lift_decode_step(data, a, b)
    
    return data

def lift_encode_face(data, a, b):
    """
    Encodes 2D face data.
    Assumes values at A are bilinear interpolations of neighbor o vertices.
    """
    # Ensure values at A are computed only from o vertices
    # Apply W-liftencode(a, b)
    data = w_lift_encode_step(data, a, b)
    
    return data

def lift_encode_edge(data, a, b):
    """
    Encodes 1D edge data.
    Assumes values at a are linear interpolations of neighbor o vertices.
    """
    # Ensure values at a are computed only from o vertices
    # Apply masks (1) and (2) from Section 4
    data = apply_mask_1(data)
    data = apply_mask_2(data)
    
    return data
```

## Operating characteristics
*   **Complexity:** Not given in the sources.
*   **Performance:** The method was applied to numerically simulated hydrodynamics data, specifically a 3D time-varying simulation of the Richtmyer-Meshkov instability and turbulent mixing in a shock tube experiment [S1, unofficial]. The dataset consisted of 274 time steps, each with an associated $20^3$ rectilinear grid, storing entropy values between 0 and 255 per vertex [S1, unofficial].
*   **Strengths:** Automatic handling of boundary cases (volumes, faces, edges) without additional examination [S1, unofficial].
*   **Limitations:** The specific numerical values for parameters $a$ and $b$ are not explicitly defined, relying on interpolation assumptions [S1, unofficial].

## Background
*   **Origin:** The method is described in the paper "Hierarchical Representation of Time-Varying Volume Data with Fourth-Root-of-Two Subdivision and Quadrilinear B-Spline Wavelets" [S1, unofficial].
*   **History:** It is part of a broader effort to compress time-varying volume data using fourth-root-of-two subdivision and B-spline wavelets [S1, unofficial].
*   **Usage:** Applied to hydrodynamics simulation data [S1, unofficial].

## Sources
[S1] Hierarchical Representation of Time-Varying Volume Data with Fourth-Root-of-Two Subdivision and Quadrilinear B-Spline Wavelets, https://digital.library.unt.edu/ark:/67531/metadc1398031/?q=volume data compression rendering, fetched 2026-10-28 (unofficial)
