# Pyramid Level Model Selection

Also called: Native vs Legacy Pyramid Policy

## Overview
Pyramid Level Model Selection determines the level-of-detail (LOD) hierarchy model for streaming volume bricks based on dataset metadata and a user-defined policy [S1, unofficial]. It solves the problem of ensuring that volume rendering pipelines can correctly interpret the spatial resolution of pre-computed or on-the-fly generated brick hierarchies, particularly when the data does not conform to strict power-of-two dimensions. It sets itself apart from general LOD selection methods by focusing specifically on the *structural definition* of the pyramid (native per-axis factors vs. a uniform virtual pyramid) rather than the runtime traversal or culling of those levels [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A dataset metadata object containing the original volume dimensions and the dimensions of each pyramid level; a policy string parameter (`'native'`, `'legacy'`, or `undefined`).
*   **Output:** A pyramid structure (list of level dimensions) or an `undefined` value to trigger a fallback mechanism.

**Data Structures**
The metadata typically contains:
*   `original_dims`: Array of 3 integers $[W, H, D]$ representing the full-resolution volume.
*   `levels`: Array of objects or arrays, where each element represents a pyramid level's dimensions $[w_i, h_i, d_i]$.

**Steps**
1.  **Check Policy:** Evaluate the `policy` parameter provided by the user or configuration.
2.  **Native Path:** If `policy` is `'native'`, return the `metadata.pyramid` directly. This assumes the stored dimensions are accurate and usable for rendering, even if they are not exact powers of two relative to the parent.
3.  **Legacy/Fallback Path:** If `policy` is `'legacy'` or `undefined` (unset), return `undefined`.
4.  **Fallback Trigger:** The rendering engine detects the `undefined` return and constructs a "uniform 2:1 virtual pyramid." This virtual pyramid ignores the stored metadata dimensions and instead calculates levels by repeatedly dividing the original dimensions by 2 (using ceiling or floor as per the engine's convention, typically $\lceil N/2 \rceil$).

**Parameters**
*   `pyramid`: String enum.
    *   `'native'`: Use per-axis factors from metadata. [S1, unofficial]
    *   `'legacy'`: Force uniform 2:1 virtual pyramid. [S1, unofficial]
    *   `undefined`/`unset`: Automatic fallback to legacy if native fails. [S1, unofficial]

**Edge Cases and Pitfalls**
*   **Native Model Failure:** The native model fails if a coarse level is slightly smaller than half its parent's dimensions (e.g., due to rounding errors during generation or non-power-of-two original sizes). In such cases, the native pyramid may be invalid for renderers expecting strict 2:1 subsampling ratios. This triggers the legacy fallback unless explicitly disabled. [S1, unofficial]
*   **Precision:** When constructing the virtual pyramid, the choice between floor and ceiling division affects the final resolution. The sources imply a standard 2:1 reduction, but exact rounding behavior should match the downsampling implementation used to generate the bricks.

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def select_pyramid_model(metadata: dict, policy: str = None) -> list or None:
    """
    Selects the pyramid level model for volume streaming.
    
    Args:
        metadata: Dict containing 'pyramid' key with list of level dimensions.
        policy: 'native', 'legacy', or None.
        
    Returns:
        List of level dimensions if native is valid/selected.
        None if fallback to virtual 2:1 pyramid is required.
    """
    # Step 1: Check policy
    if policy == 'native':
        # Step 2: Native Path
        # Return the metadata's pyramid directly
        # [G] Ensure metadata.pyramid is a deep copy to prevent mutation
        return list(metadata['pyramid'])
    
    elif policy == 'legacy':
        # Step 3: Legacy Path - Explicitly request fallback
        return None
        
    else:
        # Step 4: Unset/Default Path - Attempt native, but sources indicate
        # 'unset' triggers fallback to uniform 2:1 virtual pyramid.
        # Note: The source implies 'unset' behaves like 'legacy' in returning
        # undefined to trigger the fallback.
        return None

def generate_virtual_pyramid(original_dims: list) -> list:
    """
    Generates a uniform 2:1 virtual pyramid if native selection fails.
    This is the fallback mechanism triggered when select_pyramid_model returns None.
    """
    pyramid = [original_dims]
    current_dims = original_dims
    
    while any(d > 1 for d in current_dims):
        # Calculate next level by dividing by 2
        # [G] Using ceiling division as is common in brick-based rendering
        # to ensure coverage, though floor is also possible depending on implementation.
        next_dims = [max(1, (d + 1) // 2) for d in current_dims]
        pyramid.append(next_dims)
        current_dims = next_dims
        
    return pyramid
```

## Subtopics

### [Composite Voxel Manager with Multi-resolution Selection](18732-composite-voxel-manager-with-multi-resolution-selection.md)
Resolves texture limit failures by maintaining multiple representations (sub-resolution, sub-region, server-side bricks) and selecting the optimal one based on resolution ceilings and region coverage.

### [Partial Diamond Mesh Volume](18964-partial-diamond-mesh-volume.md)
A sparse representation encoding only relevant diamonds from a hierarchical mesh, clustering them into "supercubes" to exploit spatial coherence.

## Operating characteristics
*   **Time Complexity:** $O(1)$ for the selection logic itself. $O(L)$ for generating the virtual pyramid if fallback is triggered, where $L$ is the number of levels ($\log_2(\max(W,H,D))$).
*   **Memory Complexity:** $O(1)$ additional memory for the selection logic. The virtual pyramid generation requires storing the list of dimensions, which is negligible compared to the volume data.
*   **Strengths:** Provides a robust fallback mechanism for datasets that do not strictly adhere to power-of-two dimensions, preventing rendering failures.
*   **Limitations:** The "native" policy may fail on irregularly sized volumes, forcing a fallback that might not perfectly match the pre-computed brick data if the downsampling method differs.
*   **When to use:** Use `'native'` when the volume data was generated with a specific, non-uniform downsampling strategy that must be preserved. Use `'legacy'` or `unset` for standard power-of-two volumes or when robustness is prioritized over exact metadata fidelity.

## Background
This method originates from the implementation details of web-native, out-of-core volume rendering engines, specifically those handling multi-GB datasets in browser environments [S1, unofficial]. It addresses the practical challenges of streaming volume data where metadata accuracy and renderer expectations may diverge.

## Sources
[S1] Show HN: Kiln – WebGPU-native out-of-core volume rendering for multi-GB datasets, https://github.com/MPanknin/kiln-render, fetched 2026-10-26 (unofficial)
