# CakES MCS Layout

Also called: CakES, Cake View, MCS View

## Overview
CakES MCS Layout is a 3D visualization metaphor for Minimal Cut Sets (MCSs) in fault tree analysis, representing MCSs as cylindrical "cakes" arranged in concentric rings and levels based on their Fault Probability (FP) [S1, book]. It solves the problem of visualizing hierarchical probabilistic dependencies in complex systems by mapping abstract logical structures into a spatial layout where proximity and height encode risk and composition [S1, book]. It sets itself apart from general volume rendering methods by being a structural, glyph-based layout algorithm specifically designed for fault tree analysis, decoupling the geometric placement of MCSs from the internal structure of their constituent Basic Events (BEs) [S1, book].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of Minimal Cut Sets (MCSs). Each MCS has a computed Fault Probability (FP) and a list of Basic Events (BEs). The number of BEs in an MCS is denoted as $n$.
*   **Output:** A 3D layout where each MCS is represented as a cylinder (cake) positioned at specific coordinates $(x, y, z)$ and height $h$.

**Data Structures:**
*   **MCS Object:** Contains a list of BEs, the computed FP, and the number of BEs $n$.
*   **Level:** A grouping of MCSs with similar FP ranges.
*   **Ring:** A concentric circle within a level, indexed by `RingNo`.

**Parameters:**
*   $R_s$: Base radius of an MCS plus empty distance. Fixed at $18.5$ (derived from source text `0*.5 + 1 8`, interpreted as $0.5 + 18$) [S1, book].
*   `RingNo`: Integer index of the ring (1-based or 0-based, implied by multiplication) [S1, book].
*   $h$: Height of the MCS cylinder in the y-direction [S1, book].

**Steps:**
1.  **Compute Fault Probability:** For each MCS, compute its FP as the product of the FPs of its constituent BEs [S1, book].
2.  **Determine Levels:** Group MCSs into levels based on their FP values. The user can manipulate level boundaries using menu sliders [S1, book].
3.  **Compute Radii:**
    *   Calculate the base radius $R_s = 18.5$ [S1, book].
    *   For each ring in a level, calculate the ring radius $R_c$ using:
        $$R_c = R_s \times \text{RingNo}$$
    *   Calculate the level radius $R_h$ using:
        $$R_h = R_s + R_c$$
    [S1, book]
4.  **Calculate MCS Counts:**
    *   Calculate the number of MCSs per ring $m_r$. The source provides the formula:
        $$m_r = \max(\text{NoMCS}) = \frac{\text{length}}{\text{width}} \times \frac{\text{height}}{\text{width}} \times \frac{\text{width}}{\text{length}}$$
        *Note: The source text also mentions `π arcsin Rs Rc` in a garbled context, but the explicit formula given is the fraction product. The angular distribution logic is implied but not fully defined in the provided text.* [S1, book]
    *   Calculate the number of MCSs per level $m_h$ by summing $m_r$ for all rings in that level:
        $$m_h = \sum_{r \in \text{ring}} m_r$$
    *   Calculate the total number of MCSs $m$ by summing $m_h$ for all levels:
        $$m = \sum_{h \in \text{levels}} m_h$$
    [S1, book]
5.  **Place MCSs:**
    *   Each MCS is represented as a cylinder.
    *   The radius of the cylinder is determined by $R_s$.
    *   The height $h$ in the y-direction is determined by the number of BEs $n$ in the MCS.
    *   Placement of BEs within the MCS cylinder depends on $n$:
        *   If $n = 1$, the BE is placed in the center of the MCS $(x, y, z)$.
        *   If $n = 2$, BEs are placed in a specific configuration (details cut off in source, but implied to be symmetric) [S1, book].
    *   MCSs are distributed within the rings defined by $R_c$ and $R_h$.
    [S1, book]
6.  **Visual Enhancements:**
    *   A black ring is placed around each group of MCSs to improve distinguishability and reduce color contrast errors [S1, book].
    *   Coloring is applied based on risk level (Low, Medium, High) with specific colors for normal vision and color vision deficiency (Protanopia, Deuteranopia, Tritanopia) [S1, book].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_cakes_layout(mcs_list, rs_base=18.5):
    """
    Computes the 3D layout for CakES MCS visualization.
    
    Args:
        mcs_list: List of MCS objects, each with 'fps' (list of BE probabilities) and 'bes' (list of BEs).
        rs_base: Base radius Rs (default 18.5).
        
    Returns:
        List of MCS layout objects with 'x', 'y', 'z', 'h', 'ring_no', 'level_idx'.
    """
    # Step 1: Compute Fault Probability for each MCS
    for mcs in mcs_list:
        # FP is product of BE FPs
        mcs.fp = 1.0
        for be_fp in mcs.fps:
            mcs.fp *= be_fp
        mcs.n = len(mcs.bes)
        
    # Step 2: Group into Levels based on FP
    # Note: Exact binning logic depends on user sliders, assumed pre-sorted or binned here
    levels = group_mcs_by_fp(mcs_list) 
    
    layouts = []
    
    for level_idx, level_mcs in enumerate(levels):
        # Step 3: Determine Rings for this level
        # Assume rings are determined by some capacity or user setting
        # For simplicity, assume all MCS in a level are in one ring or distributed
        # The source implies multiple rings per level. 
        # We need to distribute MCSs into rings.
        
        # Calculate max MCSs per ring mr
        # Using the formula from source: mr = (length/width) * (height/width) * (width/length)
        # This formula seems to simplify to 1 or is context-dependent. 
        # We will use a placeholder for ring capacity if not explicitly defined by geometry.
        # However, the source says mr = max(NoMCS) = ...
        # Let's assume a fixed capacity or derived from geometry.
        # Since the formula is garbled, we use a generic capacity C_r for implementation.
        ring_capacity = 10 # Placeholder, source formula unclear
        
        rings = []
        current_ring_mcs = []
        
        for mcs in level_mcs:
            current_ring_mcs.append(mcs)
            if len(current_ring_mcs) >= ring_capacity:
                rings.append(current_ring_mcs)
                current_ring_mcs = []
        if current_ring_mcs:
            rings.append(current_ring_mcs)
            
        # Step 4: Calculate Radii and Place MCSs
        for ring_no, ring_mcs in enumerate(rings):
            # RingNo is 1-based for calculation
            ring_no_calc = ring_no + 1
            
            # Rc = Rs * RingNo
            rc = rs_base * ring_no_calc
            
            # Rh = Rs + Rc
            rh = rs_base + rc
            
            # Distribute MCSs in the ring
            # Assuming uniform angular distribution
            num_mcs_in_ring = len(ring_mcs)
            angle_step = 2 * math.pi / num_mcs_in_ring
            
            for i, mcs in enumerate(ring_mcs):
                angle = i * angle_step
                
                # Position in x-z plane
                # Radius from center is Rc? Or between Rs and Rh?
                # Source says "radius of the ring holding the MCS Rc is Rc = Rs * RingNo"
                # And "level radius Rh is Rh = Rs + Rc"
                # It implies the MCS is placed at radius Rc from the center of the level?
                # Or Rc is the radius of the ring itself.
                # Let's assume MCS center is at distance Rc from level center.
                
                x = rc * math.cos(angle)
                z = rc * math.sin(angle)
                y = 0 # Base y
                
                # Height h based on n
                # Source: "h be the height of the MCS's cylinder in y direction"
                # And "If n=1... If n=2..."
                # Let's assume h is proportional to n or fixed.
                # Source doesn't give explicit h(n) formula, only placement of BEs.
                # We'll set h = n * unit_height
                unit_height = 1.0
                h = mcs.n * unit_height
                
                layouts.append({
                    'mcs': mcs,
                    'x': x,
                    'y': y,
                    'z': z,
                    'h': h,
                    'ring_no': ring_no_calc,
                    'level_idx': level_idx
                })
                
    return layouts

def group_mcs_by_fp(mcs_list):
    """Placeholder for FP-based binning."""
    # Sort by FP
    sorted_mcs = sorted(mcs_list, key=lambda m: m.fp)
    # Simple equal binning for demonstration
    # In reality, this is controlled by user sliders
    if not sorted_mcs:
        return []
    # Assume 3 levels for example
    chunk_size = len(sorted_mcs) // 3
    levels = []
    for i in range(0, len(sorted_mcs), chunk_size):
        levels.append(sorted_mcs[i:i+chunk_size])
    return levels
```

**Edge Cases and Pitfalls:**
*   **Empty Levels:** If a level contains no MCSs, it should not be displayed [S1, book].
*   **Color Vision Deficiency:** Colors must be chosen to avoid confusion for Protanopia, Deuteranopia, and Tritanopia. Red and Green should not be neighbors [S1, book].
*   **Garbled Formulas:** The formula for $m_r$ is ambiguous in the source. Implementers should verify the intended geometric distribution logic [S1, book].
*   **BE Placement:** The exact placement of BEs within the MCS cylinder for $n > 2$ is not fully detailed in the source [S1, book].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of MCSs for placement, assuming constant ring capacity. Sorting by FP is $O(N \log N)$.
*   **Memory Complexity:** Linear with respect to the number of MCSs.
*   **Strengths:** Intuitive representation of hierarchical risk; supports color-blind users; interactive manipulation of levels.
*   **Limitations:** Limited to fault tree analysis; scalability depends on the number of MCSs and rings; formulas for ring capacity are unclear.
*   **When to Use:** Visualizing fault trees with a moderate number of MCSs where probabilistic hierarchy is key.
*   **When Not to Use:** Large-scale fault trees with thousands of MCSs may suffer from occlusion and clutter.

## Background
CakES MCS Layout was introduced in the context of fault tree analysis visualization. It is described in the DFU Volume 2, Scientific Visualization: Interactions, Features, Metaphors [S1, book]. The method aims to provide a 3D metaphor for Minimal Cut Sets, using "cakes" to represent MCSs.

## Sources
[S1] DFU, Volume 2, Scientific Visualization: Interactions, Features, Metaphors, Complete Volume, https://doi.org/10.4230/dfu.vol2.sciviz.2011, fetched 2026-10-26 (book)
