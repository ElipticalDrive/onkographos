# Layered Operator

Also called: Layering operator, L operator

## Overview
The Layered Operator is a unary SDF operation that embeds a set of existing phases $\mathcal{A}_i$ within a new outer layer $\mathcal{A}_{n+1}$ of specified thickness $t$, creating a distinct new phase [S1]. It generalizes standard SDF offsetting to multiphase environments by ensuring the new layer surrounds the union of all underlying phases without penetrating them [S1]. Unlike the Under Operator which modifies existing phase boundaries, this operator strictly adds a new phase index $n+1$ [S1].

## How it works
**Inputs:**
- A set of $n$ existing phase functions $\{\phi_{\mathcal{A}_1}, \dots, \phi_{\mathcal{A}_n}\}$, where $\phi_{\mathcal{A}_i}(\mathbf{x})$ is the signed distance to phase $i$.
- Thickness parameter $t > 0$.
- Optional blending radius $r$ for smooth union mode.

**Outputs:**
- The original $n$ phase functions unchanged.
- A new phase function $\phi_{\mathcal{A}_{n+1}}$ representing the layer.

**Data Structures:**
- Phase functions are typically stored as scalar fields or evaluated on the fly. No specific bit-packing is mandated by the definition, but standard float32 or float16 precision is implied for SDF values [S1].

**Steps:**
1. Compute the signed distance to the union of all existing phases, denoted $\phi_{\mathcal{A}_0}$. By definition, $\mathcal{A}_0$ is the complement of the union of all phases $\mathcal{A}_i$ ($i \in [1, n]$). The distance function is:
   $$ \phi_{\mathcal{A}_0}(\mathbf{x}) = -\min_{i \in [1, n]} \phi_{\mathcal{A}_i}(\mathbf{x}) $$
   This value is negative inside the union of existing phases and positive outside [S1].

2. Determine the mode of operation:
   - **Normal Mode:** Use standard union.
   - **Smooth Mode:** Use smooth union with radius $r$.

3. Compute the new layer phase function $\phi_{\mathcal{A}_{n+1}}$:

   **Normal Mode:**
   The layer is defined as the offset of the complement of the existing union, intersected with the complement itself:
   $$ \mathcal{A}_{n+1} = I(\overline{\mathcal{A}}_0, t) \cap \mathcal{A}_0 $$
   The resulting phase function is:
   $$ \phi_{\mathcal{A}_{n+1}}(\mathbf{x}) = \max(-\phi_{\mathcal{A}_0}(\mathbf{x}) - t, \phi_{\mathcal{A}_0}(\mathbf{x})) $$
   Here, $I(\cdot, t)$ represents the offsetting operator. The term $-\phi_{\mathcal{A}_0} - t$ represents the offset surface of the existing geometry, and $\phi_{\mathcal{A}_0}$ ensures the layer does not penetrate the existing phases (intersection with $\mathcal{A}_0$) [S1].

   **Smooth Mode:**
   To obtain a smooth coating effect, replace the standard union with a smooth union $\cup^*$:
   $$ \mathcal{A}_{n+1} = I\left(\bigcup_{i \in [1, n]}^* \mathcal{A}_i, t\right) \cap \mathcal{A}_0 $$
   The smooth union of two phases $i$ and $j$ with blending radius $r$ is defined as [S1]:
   $$ \phi_{\mathcal{A}_i \cup^* \mathcal{A}_j} = \min(\phi_{\mathcal{A}_i}, \phi_{\mathcal{A}_j}) - \frac{1}{6} r \left( \max\left(1 - \frac{|\phi_{\mathcal{A}_i} - \phi_{\mathcal{A}_j}|}{r}, 0\right) \right)^3 $$
   For $n > 2$ phases, the smooth union is applied iteratively or via a generalized smooth min operator [S1]. The resulting $\phi_{\mathcal{A}_{n+1}}$ is computed using the same max formula as the normal mode, but with the smooth union distance replacing $\phi_{\mathcal{A}_0}$ in the offset calculation [S1].

**Parameters:**
- $t$: Thickness of the new layer. Must be positive.
- $r$: Blending radius for the smooth union operator. Must be positive.

**Edge Cases and Pitfalls:**
- If $t$ is very small relative to the precision of the SDF, the layer may disappear or exhibit artifacts.
- In smooth mode, if $r$ is too large, the blending may extend too far, potentially causing the layer to merge visually with distant phases if not carefully bounded.
- The definition assumes $\phi_{\mathcal{A}_i}$ are valid signed distance fields. If inputs are not normalized or have incorrect signs, the min operation for $\phi_{\mathcal{A}_0}$ will fail to represent the correct union boundary [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def layered_operator(phases, thickness, smooth=False, radius=0.0):
    """
    Computes the new layer phase function for a set of existing phases.
    
    Args:
        phases: List of callable functions phi_i(x) returning signed distance.
        thickness: Float, thickness of the new layer.
        smooth: Boolean, whether to use smooth union.
        radius: Float, blending radius for smooth union (ignored if smooth=False).
        
    Returns:
        A callable function phi_new(x) for the new layer phase.
    """
    n = len(phases)
    
    def get_phi_A0(x):
        """Compute phi_A0 = -min(phi_i) for all i in [1, n]"""
        min_phi = float('inf')
        for phi in phases:
            val = phi(x)
            if val < min_phi:
                min_phi = val
        return -min_phi

    if not smooth:
        def phi_new(x):
            phi_A0 = get_phi_A0(x)
            # phi_{n+1} = max(-phi_A0 - t, phi_A0)
            return max(-phi_A0 - thickness, phi_A0)
    else:
        # Helper for smooth union of two phases
        def smooth_union(phi_i, phi_j, r):
            min_val = min(phi_i, phi_j)
            diff = abs(phi_i - phi_j)
            blend_term = max(1.0 - diff / r, 0.0)
            return min_val - (1.0 / 6.0) * r * (blend_term ** 3)

        def get_phi_smooth_union(x):
            """Compute smooth union of all phases"""
            if n == 0:
                return float('inf')
            # Start with the first phase
            current_phi = phases[0](x)
            for i in range(1, n):
                current_phi = smooth_union(current_phi, phases[i](x), radius)
            return current_phi

        def phi_new(x):
            # In smooth mode, phi_A0 is replaced by the smooth union distance
            # Note: The source defines A_{n+1} = I(Union^* A_i, t) intersect A_0
            # However, the formula for phi_{n+1} uses phi_A0.
            # The source states: "use the smooth union operator instead of the union 
            # in the definition of A_{n+1}".
            # The definition is A_{n+1} = I(Union^* A_i, t) intersect A_0.
            # The distance to Union^* A_i is the smooth union distance.
            # The distance to A_0 is the negative of the distance to Union A_i (standard).
            # But the source formula for phi_{n+1} in smooth mode is not explicitly 
            # given as a separate equation, only the definition of A_{n+1} changes.
            # Assuming the same max structure applies but with smooth union distance 
            # for the offset part, and standard A_0 for the intersection part?
            # Actually, the source says: "A_{n+1} = I(Union^* A_i, t) intersect A_0".
            # The distance to I(Union^* A_i, t) is (distance to Union^* A_i) - t.
            # The distance to Union^* A_i is get_phi_smooth_union(x).
            # The distance to A_0 is -min(phi_i).
            # So phi_{n+1} = max( (get_phi_smooth_union(x) - t), -min(phi_i) )?
            # Wait, the source formula for normal mode is:
            # phi_{n+1} = max(-phi_A0 - t, phi_A0) where phi_A0 = -min(phi_i).
            # -phi_A0 is min(phi_i). So it is max(min(phi_i) - t, -min(phi_i)).
            # This represents the offset of the union (min(phi_i) - t) intersected with 
            # the complement (-min(phi_i)).
            
            # For smooth mode, the offset is based on smooth union.
            # The intersection is still with A_0 (standard complement).
            # So:
            phi_smooth_union = get_phi_smooth_union(x)
            phi_A0_standard = get_phi_A0(x) # -min(phi_i)
            
            # Offset of smooth union: phi_smooth_union - t
            # Intersection with A_0: max(offset, phi_A0_standard)
            return max(phi_smooth_union - thickness, phi_A0_standard)

    return phi_new
```

## Subtopics

### [Smooth Union](23055-smooth-union.md)
Defines the blending operator used in the smooth mode of the Layered Operator. Choose when a seamless, rounded transition between underlying phases is desired before layering.

### [Under Operator](23053-under-operator.md)
The parent operator that the Layered Operator is derived from. Choose Under when modifying existing phase boundaries rather than adding a new phase.

## Operating characteristics
- **Time Complexity:** $O(n)$ per evaluation point, where $n$ is the number of existing phases, as it requires evaluating all phase functions to find the minimum (or smooth union).
- **Memory Complexity:** $O(1)$ additional memory per evaluation, assuming phase functions are evaluated on the fly. If precomputed, it depends on the storage of the phase fields.
- **Strengths:** Allows incremental construction of complex multiphase objects by adding layers without modifying existing phase definitions [S1]. Efficient for creating coatings or shells around existing geometry.
- **Limitations:** The layer thickness $t$ is uniform in the normal mode. In smooth mode, the blending radius $r$ affects the shape of the layer near phase boundaries, which may require tuning.
- **When to use:** When designing objects with distinct outer coatings or shells that encapsulate multiple internal phases [S1].
- **When not to use:** When the layer needs to penetrate existing phases or when computational efficiency is critical for very large $n$ without optimization.

## Background
The Layered Operator was introduced in the context of the PhaseTree framework for multiphase signed distance fields [S1]. It is directly derived from the Under Operator and generalizes the concept of SDF offsetting to multiphase scenarios [S1]. The concept of layering shapes for complex object design was previously explored by Cutler et al. [2002], which inspired the illustration of an entremet composed of series of layers [S1]. The smooth union operator used in the smooth mode is attributed to Dekkers et al. [2004] [S1].

## Sources
[S1] The PhaseTree: Multiphase Signed Distance Fields, https://doi.org/10.1145/3811379, fetched 2026-10-26
