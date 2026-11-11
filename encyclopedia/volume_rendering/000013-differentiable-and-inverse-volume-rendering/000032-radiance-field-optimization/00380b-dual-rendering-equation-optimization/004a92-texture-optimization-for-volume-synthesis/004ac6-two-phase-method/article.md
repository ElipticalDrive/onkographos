# Two-Phase Method

Also called: Two-phase synthesis, Geometry-Appearance Two-Phase Method

## Overview
The Two-Phase Method is a patch-based synthesis technique designed to ensure both geometric and appearance continuity in synthesized volumetric textures, particularly for complex scenes with varying lighting [S1]. It addresses the problem of selecting candidate patches that satisfy multiple constraints by decoupling the selection process into two distinct stages: first filtering for geometric consistency, then refining for appearance consistency [S1]. This separation allows the method to handle complex geometry and lighting variations more effectively than single-phase approaches [S1].

## How it works
The method operates on a set of candidate patches derived from an exemplar volume. It requires a density field $V^{(d)}$ and a color feature field $V^{(f)}$ from the exemplar, along with a shading map to guide lighting consistency [S1].

**Inputs:**
- Candidate patch set $\mathcal{P}$ (filtered by shading similarity).
- Exemplar density field $V^{(d)}$.
- Exemplar color feature field $V^{(f)}$.
- Synthesis target patch $P_n$.

**Outputs:**
- Selected patch $P_{selected}$ from $\mathcal{P}$ that minimizes discontinuity errors.

**Data Structures:**
- Patches are represented as 3D voxel blocks.
- Density field $V^{(d)}$ stores scalar density values.
- Color feature field $V^{(f)}$ stores color/feature vectors.

**Steps:**
1. **Phase 1: Geometry-based Synthesis**
   - For each candidate patch $P_i \in \mathcal{P}$, calculate the geometric continuity error against the target $P_n$ using the density field $V^{(d)}$.
   - The error is typically measured by comparing the density values at the boundaries of the patches [S1].
   - Select a subset of $k_g$ candidate patches that exhibit the highest geometric continuity (lowest density discontinuity) [S1].

2. **Phase 2: Appearance-based Synthesis**
   - For each of the $k_g$ candidates from Phase 1, calculate the appearance continuity error against the target $P_n$ using the color feature field $V^{(f)}$.
   - The error is measured by comparing color features at the boundaries [S1].
   - Select the single patch $P_{selected}$ with the lowest appearance error [S1].

**Formulas:**
- Geometric Error $E_g(P_i, P_n)$: Not explicitly defined in the provided notes, but implied to be based on $V^{(d)}$ [S1].
- Appearance Error $E_a(P_i, P_n)$: Not explicitly defined in the provided notes, but implied to be based on $V^{(f)}$ [S1].

**Parameters:**
- $k_g$: Number of candidates retained after Phase 1. Not given in the sources.

**Edge Cases and Pitfalls:**
- If the candidate set $\mathcal{P}$ is empty or too small, the method may fail to find a suitable patch [S1].
- Precision issues in density or color feature comparison may lead to suboptimal selections [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def two_phase_synthesis(target_patch, candidate_patches, density_field, color_field):
    """
    Selects a patch from candidate_patches using a two-phase method.
    
    Args:
        target_patch: The patch being synthesized.
        candidate_patches: List of candidate patches.
        density_field: Exemplar density field V^(d).
        color_field: Exemplar color feature field V^(f).
        
    Returns:
        The selected patch.
    """
    # Phase 1: Geometry-based Synthesis
    geometric_errors = []
    for patch in candidate_patches:
        # Calculate geometric error based on density field V^(d)
        # Error is derived from boundary density discontinuities [S1]
        err = calculate_geometric_error(patch, target_patch, density_field)
        geometric_errors.append((patch, err))
    
    # Sort by error and select top k_g candidates
    # k_g is not specified in sources [G]
    geometric_errors.sort(key=lambda x: x[1])
    k_g = 10 # Placeholder value [G]
    top_candidates = [x[0] for x in geometric_errors[:k_g]]
    
    # Phase 2: Appearance-based Synthesis
    appearance_errors = []
    for patch in top_candidates:
        # Calculate appearance error based on color feature field V^(f)
        # Error is derived from boundary color feature discontinuities [S1]
        err = calculate_appearance_error(patch, target_patch, color_field)
        appearance_errors.append((patch, err))
    
    # Select patch with minimum appearance error
    best_patch, _ = min(appearance_errors, key=lambda x: x[1])
    return best_patch

def calculate_geometric_error(patch1, patch2, density_field):
    """
    Calculates geometric continuity error using density field.
    Implementation detail: Compare density values at shared boundaries [G].
    """
    # Not given in the sources [G]
    pass

def calculate_appearance_error(patch1, patch2, color_field):
    """
    Calculates appearance continuity error using color feature field.
    Implementation detail: Compare color features at shared boundaries [G].
    """
    # Not given in the sources [G]
    pass
```

## Subtopics

### Edge Error Factor
A variant for textures without sharp boundaries that introduces an edge error factor into the synthesis process to measure and balance average edge discontinuities [S2, unofficial]. It calculates edge errors $E_e$ by measuring average edge discontinuities for composed textures, considering that discontinuities along an edge come from 2 vertical pairs of adjacent cube faces (e.g., A(E)-C(E) and C(N)-D(N) for edge a1-b2-c3-d4) [S2, unofficial]. For each face of a cube, errors of the 4 edges are calculated from all cubes having the same color on the adjacent face, collecting the 2 face errors of the 12 edges from all combinations once and only once [S2, unofficial]. The accumulated synthesis error is $E_s = p_i \times E_i + p_e \times E_e$, where $E_i$ is the inside error, $E_e$ is the edge error, and $p_i, p_e$ are weights set as the inverse of their cutting path size to balance the errors [S2, unofficial].

## Operating characteristics
The method ensures geometric and appearance continuity, which is crucial for complex texture and geometry [S1]. It allows for relighting scenes without decoupling the scene by using shading-guided synthesis [S1]. Performance details such as time and memory complexity are not explicitly provided in the sources [S1][S2, unofficial]. The method is suitable for synthesizing large-scale scenes with reasonable lighting [S1].

## Background
The Two-Phase Method was proposed in the context of NeRF Synthesis with Shading Guidance [S1]. It is part of a pipeline that includes intrinsic decomposition, shading map extraction, and shading-guided synthesis [S1]. The method aims to solve high-dimensional texture synthesis problems inspired by traditional exemplar-based methods [S1].

## Sources
[S1] NeRF Synthesis with Shading Guidance, https://link.springer.com/chapter/10.1007/978-981-99-9666-7_16, fetched 2026-10-26
[S2] Example-based volume illustrations, https://ieeexplore.ieee.org/abstract/document/1532854/, fetched 2026-10-31 (unofficial)
