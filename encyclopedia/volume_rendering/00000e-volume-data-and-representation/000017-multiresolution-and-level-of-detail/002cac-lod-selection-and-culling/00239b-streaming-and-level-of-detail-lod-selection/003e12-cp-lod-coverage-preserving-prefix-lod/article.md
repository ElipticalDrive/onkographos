# CP-LOD (Coverage-Preserving Prefix LOD)

Also called: GPU selection, real-time rendering at city scale

## Overview
CP-LOD is a runtime selection method that traverses a factree on the GPU to produce a leaf-only cut bounded by a capacity budget $K$, using Morton-strided subsampling to preserve visual coverage [S1, unofficial]. It differs from naive threshold schedules by being resolution-invariant by construction, producing identical materialized sets at different resolutions (e.g., $800^2$ and $1920 \times 1080$) [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A factree (hierarchical representation of Gaussian primitives), a capacity budget parameter $c$, and the target rendering resolution.
*   **Output:** A standard primitive stream (means, quaternions, scales, opacities, spherical harmonics coefficients) consumable by any splatting rasterizer [S1, unofficial].

**Data Structures:**
The method operates on a factree structure. The specific internal layout of the factree nodes is not detailed in the sources, but the traversal produces a "leaf-only cut" [S1, unofficial].

**Parameters:**
*   $c$: A single parameter chosen at deploy time that determines the memory budget tier [S1, unofficial].
    *   Typical values: $c=4, 8, 16$ [S1, unofficial].
    *   Corresponding memory budgets: $c=4 \rightarrow 1288$ MB, $c=8 \rightarrow 824$ MB, $c=16 \rightarrow 407$ MB [S1, unofficial].

**Steps:**
1.  **Traverse the factree on GPU:** Perform a traversal to identify candidate nodes for the leaf-only cut [S1, unofficial].
2.  **Apply capacity budget:** Bound the materialized working set to the capacity budget $K$ determined by $c$ [S1, unofficial].
3.  **Morton-strided subsampling:** To preserve visual coverage during subsampling, apply a scale factor of $\sqrt{r}$ and compensate opacity using the formula:
    $$ \alpha_{new} = 1 - (1 - \alpha_{old})^r $$
    where $r$ is the subsampling factor [S1, unofficial].
4.  **Collect primitives:** Collect the selected prefix directly into rasterizer buffers. This step avoids CPU-side collection or host-to-device upload [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cp_lod_selection(factree, capacity_budget_c):
    """
    Selects a leaf-only cut from a factree bounded by a capacity budget.
    
    Args:
        factree: The hierarchical tree structure of Gaussian primitives.
        capacity_budget_c: Integer parameter determining memory tier (e.g., 4, 8, 16).
        
    Returns:
        list: A list of primitive attributes (means, quaternions, scales, opacities, SH)
              ready for rasterization.
    """
    # Step 1: Traverse the factree on GPU to find a leaf-only cut
    # The traversal logic is specific to the factree implementation.
    # We assume a function that returns a set of candidate leaf nodes.
    candidate_leaves = traverse_factree_gpu(factree)
    
    # Step 2: Apply capacity budget K to bound the materialized working set
    # The mapping from c to K (in MB) is defined by the deployment configuration.
    # For c=4, K=1288 MB; c=8, K=824 MB; c=16, K=407 MB.
    # The selection algorithm filters candidate_leaves to fit within K.
    selected_nodes = select_nodes_by_budget(candidate_leaves, capacity_budget_c)
    
    # Step 3: Perform Morton-strided subsampling with opacity compensation
    # This step ensures visual coverage is preserved when subsampling.
    final_primitives = []
    for node in selected_nodes:
        # Determine subsampling factor r based on the node's level or stride
        r = get_subsampling_factor(node)
        
        # Extract base attributes
        alpha_old = node.opacity
        mean = node.mean
        scale = node.scale
        rot = node.rotation
        sh = node.sh
        
        # Apply opacity compensation: alpha_new = 1 - (1 - alpha_old)^r
        alpha_new = 1.0 - (1.0 - alpha_old) ** r
        
        # Apply scale factor sqrt(r) to maintain coverage footprint
        scale_new = scale * sqrt(r)
        
        # Add compensated primitive to the output stream
        final_primitives.append({
            'mean': mean,
            'scale': scale_new,
            'rotation': rot,
            'opacity': alpha_new,
            'sh': sh
        })
        
    # Step 4: Collect the selected prefix straight into rasterizer buffers
    # This is done directly on the GPU without CPU-side collection or H2D upload.
    rasterizer_buffers = upload_to_gpu_buffers(final_primitives)
    
    return rasterizer_buffers
```

**Edge Cases and Pitfalls:**
*   **Residual popping:** Occurs at tier boundaries (transitions between different values of $c$), though it is confined to these transitions [S1, unofficial].

## Subtopics

## Operating characteristics
**Time and Memory Complexity:**
*   The method allows for interactive rendering under a capacity budget.
*   Memory usage is directly controlled by the parameter $c$ [S1, unofficial].

**Measured Performance:**
*   **Hardware:** RTX 4080 SUPER [S1, unofficial].
*   **Resolution:** 1080p [S1, unofficial].
*   **Block_all v3 retrain (6.89M primitives):**
    *   Full set: 71 fps mean [S1, unofficial].
    *   $c=4$ (1288 MB): 76.8 fps [S1, unofficial].
    *   $c=8$ (824 MB): 65.0 fps [S1, unofficial].
    *   $c=16$ (407 MB): 59.0 fps [S1, unofficial].
*   **MatrixCity Aerial model (8.56M primitives):**
    *   Quality is visually preserved down to the 2 GB tier with $|\Delta PSNR| \le 0.27$ dB vs. full render [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Resolution-invariant selection; no retraining or re-export required for different memory budgets; efficient GPU-only pipeline [S1, unofficial].
*   **Limitations:** Residual popping at tier boundaries [S1, unofficial].

**When to use:**
*   Real-time rendering at city scale where memory bandwidth is a bottleneck and visual quality must be preserved across different hardware configurations [S1, unofficial].

## Background
CP-LOD was introduced in the paper "Budgeted-GS: Real-Time Large-Scale Gaussian Splatting via Factoring LOD" [S1, unofficial]. It enables real-time rendering of large-scale Gaussian splatting scenes by factoring the LOD selection process into a GPU-traversable factree structure.

## Sources
[S1] Budgeted-GS: Real-Time Large-Scale Gaussian Splatting via Factoring LOD, https://arxiv.org/abs/2610.03162, fetched 2026-10-29 (unofficial)
