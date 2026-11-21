# History Rejection

Also called: History Validation, Stale History Discard

## Overview
History Rejection is a technique used in temporal anti-aliasing (TAA) to mitigate ghosting artifacts by discarding a portion of historical pixel data deemed invalid or stale [S1, unofficial]. It addresses the problem of temporal accumulation errors in participating media, where direct comparison between aliased current frame samples and history leads to spatial and temporal instability [S1, unofficial]. Unlike its parent method, Adaptive Blending, which modifies the blend factor to reduce ghosting, History Rejection specifically focuses on detecting and discarding invalid history to prevent it from contaminating the current frame [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Current frame sample ($C$), historical frame sample ($H$), previous blend factor ($\alpha$).
- Output: Updated blend factor ($\alpha'$) or rejection flag.

**Data Structures:**
- Pixel history buffer: Stores accumulated color/radiance values.
- Error estimate buffer: Stores filtered difference metrics between $C$ and $H$.

**Steps:**
1. **Compute Error Estimate:** Calculate the difference between the current frame sample and the historical sample. Direct comparison is unstable due to aliasing [S1, unofficial].
2. **Filter Error Estimate:** Apply a box filter to the error estimate to stabilize detection of stale history [S1, unofficial]. This reduces noise caused by jittering [S1, unofficial].
3. **Detect Stale History:** Analyze the filtered error estimate to determine if the history is stale. Staleness is indicated by drastic changes in light color or intensity, or by high variance in the neighborhood [S1, unofficial].
4. **Adjust Blend Factor ($\alpha$):**
   - **Hard Reject:** If stale history is detected, set $\alpha = 1$. This discards the history entirely, using only the current frame sample [S1, unofficial].
   - **Soft Reject:** Limit $\alpha$ to a high minimum value (e.g., $\alpha_{min}$). This ensures some history is always blended in, providing a smoother transition and preventing complete loss of temporal coherence [S1, unofficial].
   - **Adaptive Blending:** Adjust $\alpha$ based on the color extent of a neighborhood of current frame samples. If the color bounding box is small (flat color), $\alpha$ is increased to reduce ghosting. If the color range is large (sharp edge/noise), $\alpha$ is decreased to introduce temporal blurring [S1, unofficial].

**Formulas:**
- Error Estimate: $E = |C - H|$
- Filtered Error: $E_{filt} = \text{BoxFilter}(E)$
- Blend Factor Update:
  - If $E_{filt} > \text{Threshold}$: $\alpha' = 1$ (Hard Reject)
  - Else: $\alpha' = \max(\alpha_{min}, \alpha_{base})$ (Soft Reject)
  - Where $\alpha_{base}$ is adjusted based on neighborhood color extent [S1, unofficial].

**Parameters:**
- $\alpha_{min}$: Minimum blend factor for soft reject. Typical value: Not given in the sources.
- Threshold: Error threshold for hard reject. Typical value: Not given in the sources.
- Box Filter Size: Size of the box filter applied to the error estimate. Typical value: Not given in the sources.

**Edge Cases and Pitfalls:**
- **Jittering:** Sampling points may jump between high and low radiance/density regions quickly, even without camera or scene movement. This makes it difficult to distinguish dynamic scene changes from jitter artifacts [S1, unofficial].
- **Aliasing:** Direct comparison of history to current frame samples results in spatial and temporal instability because current frame samples are inevitably aliased [S1, unofficial]. Filtering is required to mitigate this [S1, unofficial].
- **High Variance Regions:** Issues are exacerbated in regions with high variance, such as near light sources or volume boundaries [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per pixel for error computation and filtering, assuming fixed-size box filter.
- **Memory Complexity:** $O(N)$ for storing error estimate buffer, where $N$ is the number of pixels.
- **Strengths:** Effectively reduces ghosting artifacts by discarding invalid history. Simple to implement.
- **Limitations:** Prone to instability if not properly filtered. May cause flickering if thresholds are not tuned correctly.
- **When to Use:** When ghosting artifacts are prominent due to stale history.
- **When Not to Use:** When performance is critical and filtering overhead is unacceptable.

## Background
History Rejection is discussed in the context of temporal anti-aliasing for real-time froxel-based volumetric rendering [S1, unofficial]. It builds on the concept of adaptive blending [S1, unofficial]. The technique is attributed to various researchers in the field, including Malan, who proposed using color range in the neighborhood to adaptively adjust the blend factor [S1, unofficial].

## Sources
[S1] Temporal Anti-Aliasing for Real-Time Froxel-Based Volumetric Rendering - Advances in volumetric temporal anti-aliasing for real-time rendering of participating media, https://openalex.org/W7218067689, fetched 2026-11-01 (unofficial)
