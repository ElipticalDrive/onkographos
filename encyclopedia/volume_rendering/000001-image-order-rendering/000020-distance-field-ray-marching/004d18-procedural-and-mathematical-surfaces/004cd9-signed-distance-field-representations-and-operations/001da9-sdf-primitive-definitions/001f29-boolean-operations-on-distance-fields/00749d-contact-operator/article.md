# Contact Operator

Also called: Phase Contact Operator, Weighted SDF Combination

## Overview
The Contact Operator produces a contact surface between two distinct phases $\mathcal{A}$ and $\mathcal{B}$ within their intersection $\mathcal{A} \cap \mathcal{B}$, simulating collision without deformation [S1]. It interpolates between the Over and Under operators defined in [Boolean Operations on Distance Fields](1961-boolean-operations-on-distance-fields.md) using a weight parameter $\alpha$, allowing for a tunable boundary between the two materials [S1].

## How it works
**Inputs:**
*   Two Signed Distance Fields (SDFs): $\phi_{\mathcal{A}_i}$ for phase $i$ of object $\mathcal{A}$, and $\phi_{\mathcal{B}_j}$ for phase $j$ of object $\mathcal{B}$.
*   Weight parameter $\alpha \in [0, 1]$.

**Outputs:**
*   Updated multiphase function $\Phi$ where phases $i$ and $j$ are modified to reflect contact, and all other phases $k \neq i, j$ are set to infinity.

**Data Structures:**
*   The method operates on scalar fields. For implementation, $\phi_{\mathcal{A}_i}$ and $\phi_{\mathcal{B}_j}$ are typically stored as 3D grids or evaluated procedurally. The output $\Phi$ is a multiphase representation where each voxel or point stores the active phase ID and its distance value.

**Steps:**
1.  **Compute the Contact Field $f$:** Calculate the 1-Lipschitz signed distance field $f(\mathbf{p})$ at every point $\mathbf{p}$ using the weighted sum of the input SDFs:
    $$f(\mathbf{p}) = \alpha \phi_{\mathcal{A}_i}(\mathbf{p}) - (1 - \alpha) \phi_{\mathcal{B}_j}(\mathbf{p})$$
2.  **Define Regions:** Identify the regions $\Omega^+$ and $\Omega^-$ based on the sign of $f(\mathbf{p})$:
    *   $\Omega^+ = \{\mathbf{p} \mid f(\mathbf{p}) > 0\}$
    *   $\Omega^- = \{\mathbf{p} \mid f(\mathbf{p}) \le 0\}$
    *   Note: $\Omega^+ \cup \Omega^- = \mathbb{R}^3$ and $\Omega^+ \cap \Omega^- = \{\mathbf{p} \mid f(\mathbf{p}) = 0\}$.
3.  **Update Phase $i$ ($\mathcal{A}$):** The new distance field for phase $i$, denoted $\phi_i$, is the maximum of the original phase distance and the negated contact field:
    $$\phi_i(\mathbf{p}) = \max(\phi_{\mathcal{A}_i}(\mathbf{p}), -f(\mathbf{p}))$$
    This effectively subtracts the region $\Omega^+$ from $\mathcal{A}$ ($\tilde{\mathcal{A}} = \mathcal{A} - \Omega^+$) [S1].
4.  **Update Phase $j$ ($\mathcal{B}$):** The new distance field for phase $j$, denoted $\phi_j$, is the maximum of the original phase distance and the contact field:
    $$\phi_j(\mathbf{p}) = \max(\phi_{\mathcal{B}_j}(\mathbf{p}), f(\mathbf{p}))$$
    This effectively subtracts the region $\Omega^-$ from $\mathcal{B}$ ($\tilde{\mathcal{B}} = \mathcal{B} - \Omega^-$) [S1].
5.  **Invalidate Other Phases:** For all other phases $k \neq i, j$, set their distance values to infinity to indicate they are not present in the contact region:
    $$\phi_k(\mathbf{p}) = \infty$$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def contact_operator(phi_A_i, phi_B_j, alpha):
    """
    Computes the contact surface between phase i of A and phase j of B.
    
    Args:
        phi_A_i: SDF function or grid for phase i of object A.
        phi_B_j: SDF function or grid for phase j of object B.
        alpha: Weight parameter in [0, 1].
        
    Returns:
        phi_i: Updated SDF for phase i.
        phi_j: Updated SDF for phase j.
    """
    # Step 1: Compute the weighted contact field f
    # f(p) = alpha * phi_A_i(p) - (1 - alpha) * phi_B_j(p)
    f = alpha * phi_A_i - (1 - alpha) * phi_B_j
    
    # Step 2 & 3: Update phase i
    # phi_i = max(phi_A_i, -f)
    # This corresponds to A - Omega^+ where Omega^+ is f > 0
    phi_i = max(phi_A_i, -f)
    
    # Step 4: Update phase j
    # phi_j = max(phi_B_j, f)
    # This corresponds to B - Omega^- where Omega^- is f <= 0
    phi_j = max(phi_B_j, f)
    
    # Step 5: Other phases are set to infinity (not returned here, 
    # but implied in the multiphase structure)
    
    return phi_i, phi_j
```

**Parameters:**
*   $\alpha \in [0, 1]$: Weight parameter.
    *   $\alpha \to 0$: $f \to \phi_{\mathcal{B}_j}$, $\Omega^+ \to \Omega_{\mathcal{B}}$. The operator approaches the Under operator [S1].
    *   $\alpha \to 1$: $f \to \phi_{\mathcal{A}_i}$, $\Omega^- \to \Omega_{\mathcal{A}}$. The operator approaches the Over operator [S1].
    *   $\alpha = 1/2$: The implicit surface $\partial\Omega_\alpha = \{\mathbf{p} \mid f(\mathbf{p}) = 0\}$ separates space into domains where $\phi_{\mathcal{A}_i} > \phi_{\mathcal{B}_j}$ and $\phi_{\mathcal{A}_i} < \phi_{\mathcal{B}_j}$ [S1].

**Edge Cases:**
*   **Precision:** The boundary is defined by $f(\mathbf{p}) = 0$. Floating-point precision errors near this zero-crossing may cause artifacts in the contact surface.
*   **Non-intersecting Phases:** If $\mathcal{A}$ and $\mathcal{B}$ do not intersect, the behavior is determined by the global SDF values, but the "contact" surface may not be visually meaningful as a collision interface.

## Subtopics

### [FRep R-function Boolean Operations](19951-frep-r-function-boolean-operations.md)
Uses smooth R-functions for Boolean operations. Choose when smooth blending of shapes is required rather than distinct phase contact.

### [Under Operator (Non-commutative Union)](23053-under-operator-non-commutative-union.md)
A specific non-commutative union. Choose when one object must strictly occlude another without a shared contact interface.

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of points/voxels, as it requires a single pass to compute $f$ and two max operations.
*   **Memory Complexity:** $O(N)$ to store the intermediate field $f$ and the output fields.
*   **Strengths:** Provides a tunable interface between two phases, allowing for a smooth transition or specific collision boundary defined by $\alpha$. It avoids the deformation complexity of earlier contact models [S1].
*   **Limitations:** Only applies to two specific phases at a time. Other phases are invalidated ($\infty$), which may require careful management in complex multiphase scenes.
*   **When to use:** When simulating contact between two distinct materials (e.g., rock and cement) where a specific boundary surface is desired within their overlap [S1].

## Background
The Contact Operator is described in *The PhaseTree: Multiphase Signed Distance Fields* [S1]. It draws inspiration from the work of Gascuel et al. [1993] but focuses on collision without deformation [S1]. It is used in multiphase SDF representations, such as synthesizing a rock wall by combining Voronoi cells (rock blocks) with a cement phase [S1].

## Sources
[S1] The PhaseTree: Multiphase Signed Distance Fields, https://doi.org/10.1145/3811379, fetched 2026-10-26
