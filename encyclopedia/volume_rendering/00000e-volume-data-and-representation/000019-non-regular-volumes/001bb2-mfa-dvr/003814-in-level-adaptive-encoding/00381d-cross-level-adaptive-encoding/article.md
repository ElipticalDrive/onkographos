# Cross-level Adaptive Encoding

Also called: Hierarchical Adaptive Encoding

## Overview
Cross-level Adaptive Encoding is a hierarchical optimization technique that reduces the computational cost of determining the optimal Number of Control Points (NCP) for micro-blocks in multi-resolution volume rendering [S1, unofficial]. It leverages lower LOD results to avoid exhaustive in-level searches for higher LODs, reducing overall encoding time [S1, unofficial]. It builds on [In-level Adaptive Encoding](8165-in-level-adaptive-encoding.md) by using its results from lower LODs to constrain or skip searches in higher LODs [S1, unofficial].

## How it works
**Inputs:**
- A set of LODs ranging from $LOD_{low}$ to $LOD_{high}$ [S1, unofficial].
- The polynomial degree $degree$ used for the Functional Approximation (MFA) [S1, unofficial].
- An error bound threshold [S1, unofficial].
- The maximum possible NCP value $n$ [S1, unofficial].

**Outputs:**
- For each micro-block at each LOD, the optimal NCP ($NCP^*$) and the set of complex micro-blocks ($ComplexSet$) [S1, unofficial].

**Data Structures:**
- $ComplexSet_{LOD}$: A set or boolean mask identifying micro-blocks at a specific LOD that require expensive encoding (i.e., $NCP^* > degree+2$) [S1, unofficial].
- $errs$: A temporary list storing the approximation error for each candidate NCP during the search [S1, unofficial].

**Definitions:**
- **Simple Micro-block:** A micro-block where the optimal NCP is the minimum possible value, defined as $degree+2$ [S1, unofficial]. These correspond to regions with constant values or values with very small deviations [S1, unofficial].
- **Complex Micro-block:** A micro-block where the optimal NCP is greater than $degree+2$ [S1, unofficial]. These correspond to regions with more dynamic values [S1, unofficial].

**Algorithm Steps:**
1. **Initialize Lowest LOD:** Perform standard in-level adaptive encoding on $LOD_{low}$ to establish baseline $NCP^*$ and error mappings [S1, unofficial].
2. **Identify Complex Blocks at Lowest LOD:** For each micro-block in $LOD_{low}$, if its optimal $NCP^* > degree+2$, add it to $ComplexSet_{LOD_{low}}$ [S1, unofficial].
3. **Iterate Higher LODs:** Loop through each LOD from $LOD_{low} + 1$ to $LOD_{high}$ [S1, unofficial].
4. **Check Coverage:** For each micro-block in the current LOD, determine if it is spatially covered by any micro-block in $ComplexSet_{LOD-1}$ [S1, unofficial].
5. **Encode Complex Regions:** If the micro-block is covered by a complex block from the previous LOD:
   - Initialize an empty list $errs$ [S1, unofficial].
   - Iterate through candidate NCP values from $n$ down to $degree+2$ [S1, unofficial].
   - For each candidate NCP, compute the approximation error $err$ using the model function $model(NCP, degree)$ [S1, unofficial].
   - Append $err$ to $errs$ [S1, unofficial].
   - If $err > error\_bound$, mark the micro-block as complex and break the search [S1, unofficial].
   - Find $NCP^*$ from $errs$ to encode the micro-block using $NCP^* = \arg \min_{NCP} (Error(NCP) < error\_bound)$ [S1, unofficial].
   - If the resulting $NCP^* > degree+2$, add the current micro-block to $ComplexSet_{LOD}$ [S1, unofficial].
6. **Encode Simple Regions:** If the micro-block is *not* covered by any complex block from the previous LOD:
   - Assume the region is simple [S1, unofficial].
   - Set $NCP^* = degree+1$ to encode the micro-block [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cross_level_adaptive_encoding(LODs, degree, error_bound, max_n):
    # LODs is a list of LOD objects, indexed from 0 (lowest) to N (highest)
    # Each LOD object has a list of micro_blocks, each with a parent_lod_id attribute
    
    # Step 1: Process lowest LOD
    lowest_lod = LODs[0]
    complex_set_prev = set()
    
    for mb in lowest_lod.micro_blocks:
        errs = []
        for ncp in range(max_n, degree + 1, -1):
            # Evaluate MFA approximation error for this micro-block using ncp control points
            err = abs(mb.data - mb.estimate(ncp, degree)).max()
            errs.append(err)
            if err > error_bound:
                is_complex = True
                break
        else:
            is_complex = False
            
        # Find smallest NCP such that error < error_bound
        # errs[0] corresponds to max_n, errs[-1] corresponds to degree+2
        optimal_ncp = degree + 2
        for i, err in enumerate(errs):
            if err > error_bound:
                optimal_ncp = max_n - (i - 1) if i > 0 else degree + 2
                break
                
        mb.ncp = optimal_ncp
        
        if is_complex or optimal_ncp > degree + 2:
            complex_set_prev.add(mb.id)
            
    # Step 2: Process higher LODs
    for lod_idx in range(1, len(LODs)):
        current_lod = LODs[lod_idx]
        complex_set_curr = set()
        
        for mb in current_lod.micro_blocks:
            # Check if parent at previous LOD is in the complex set
            is_covered = mb.parent_lod_id in complex_set_prev
            
            if is_covered:
                errs = []
                for ncp in range(max_n, degree + 1, -1):
                    err = abs(mb.data - mb.estimate(ncp, degree)).max()
                    errs.append(err)
                    if err > error_bound:
                        is_complex = True
                        break
                else:
                    is_complex = False
                    
                optimal_ncp = degree + 2
                for i, err in enumerate(errs):
                    if err > error_bound:
                        optimal_ncp = max_n - (i - 1) if i > 0 else degree + 2
                        break
                        
                mb.ncp = optimal_ncp
                
                if is_complex or optimal_ncp > degree + 2:
                    complex_set_curr.add(mb.id)
            else:
                # Skip exhaustive search, use default for simple blocks
                mb.ncp = degree + 1
                
        complex_set_prev = complex_set_curr
```

**Parameters:**
- $degree$: The polynomial degree of the MFA basis functions [S1, unofficial].
- $error\_bound$: The threshold for the maximum acceptable approximation error [S1, unofficial].
- $max\_n$: The highest possible NCP value to start the search from [S1, unofficial].

**Edge Cases and Pitfalls:**
- The method relies on the assumption that complex micro-blocks from a lower LOD always cover the complex micro-blocks of a higher LOD [S1, unofficial]. If this assumption fails, simple regions might be incorrectly encoded with higher complexity, or complex regions might be skipped [S1, unofficial].
- The pseudocode explicitly assigns $NCP^* = degree+1$ for uncovered micro-blocks, while the textual definition classifies simple micro-blocks as having $NCP = degree+2$ [S1, unofficial]. Implementations should follow the pseudocode assignment for the skip path [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Reduces the average encoding time compared to performing in-level adaptive encoding on all micro-blocks at all LODs [S1, unofficial]. The complexity depends on the ratio of complex to simple micro-blocks [S1, unofficial].
- **Memory Complexity:** Requires storing the $ComplexSet$ for each LOD, which is proportional to the number of micro-blocks [S1, unofficial].
- **Strengths:** Significantly speeds up the encoding process for large-scale volumes by avoiding redundant exhaustive searches in smooth regions [S1, unofficial].
- **Limitations:** Relies on the assumption that complex micro-blocks from a lower LOD always cover the complex micro-blocks of a higher LOD [S1, unofficial].

## Background
Cross-level Adaptive Encoding was introduced in the context of Adaptive Multi-Resolution Encoding for Interactive Large-Scale Volume Visualization through Functional Approximation [S1, unofficial]. It addresses the bottleneck of encoding time in multi-resolution volume rendering pipelines by exploiting the hierarchical nature of data complexity [S1, unofficial].

## Sources
[S1] Adaptive Multi-Resolution Encoding for Interactive Large-Scale Volume ..., https://www.researchgate.net/publication/383700795_Adaptive_Multi-Resolution_Encoding_for_Interactive_Large-Scale_Volume_Visualization_through_Functional_Approximation, fetched 2026-10-30 (unofficial)
