# Variable Convolution Length for Dye
Also called: Separate dye convolution length

## Overview
Variable Convolution Length for Dye is a technique that decouples the convolution length of the dye advection from the global Line Integral Convolution (LIC) texture [S1, unofficial]. It solves the problem where extending the global convolution length to highlight long-range flow features degrades the contrast of the LIC texture and increases computational cost [S1, unofficial]. By allowing a separate, longer convolution length for the dye, it enables long dye trails without compromising the efficiency or visual quality of the underlying LIC [S1, unofficial].

## How it works
**Inputs:**
- Vector field defining the flow.
- Scalar noise field for LIC texture.
- Global convolution length $L$.
- Dye convolution length $T$.
- Standard LIC animation steps $N_{LIC}$.

**Outputs:**
- Final composite scalar field combining LIC texture and dye advection.

**Data Structures:**
- Standard LIC output buffer $F_{out}$ of size $N_{LIC} \times W \times H$.
- Dye advection output buffer $D_{out}$ of size $X \times W \times H$, where $X$ is the number of dye steps.
- Input dyed texture $D_{in}$.

**Parameters:**
- $L$: Global convolution length.
- $T$: Dye convolution length.
- $N_{LIC}$: Number of steps for standard LIC.
- Assumption: $T$ is a multiple of $L$ [S1, unofficial].

**Steps:**
1. Calculate the number of animation steps required for the dye advection $X$:
   $$ X = N_{LIC} \times \frac{T}{L} $$
   [S1, unofficial]

2. Compute the standard LIC output $F_{out}$ for $N_{LIC}$ steps. For cells not affected by the dye, only these $N_{LIC}$ steps are computed [S1, unofficial].

3. Perform dye advection for $X$ steps. For each step $I \in [0, X-1]$ and cell $(x,y)$, compute the dyed output $D_{out}(I,x,y)$ using the revised LIC formula:
   $$ D_{out}(I,x,y) = \frac{\sum_{i=0}^{Z-1} D_{in}(P_i) \cdot \mathbb{I}(S_i \in [r, r + \Delta L])}{\sum_{i=0}^{Z-1} \mathbb{I}(S_i \in [r, r + \Delta L])} $$
   Where:
   - $D_{in}(P_i)$ is the input dyed texture value at position $P_i$.
   - $D_{out}(I,x,y)$ is the output value for the dyed cell $(x,y)$ at step $I$.
   - $I \in [0, Z-1]$ is the current animation step (Note: Source uses $Z$ as the upper bound for the summation index, likely corresponding to the number of samples or steps in the filter, distinct from the total animation steps $X$) [S1, unofficial].
   - $\Delta L$ is the length of the filter box [S1, unofficial].
   - $\mathbb{I}(\cdot)$ is an indicator predicate: 1 if $S_i \in [r, r + \Delta L]$, 0 otherwise. This predicate determines if the sample point $S_i$ falls within the dye filter range [S1, unofficial].
   - Note: The source text contains garbled characters in the formula representation. The structure implies a weighted average of the dye texture along the streamline segment defined by the filter box, restricted to the dye's influence range [S1, unofficial].

4. Combine the standard LIC output with the dye advection output. For each frame or final output, combine $D_{out}$ and $F_{out}$:
   $$ Final_{out}(I, x, y) = D_{out}(I, x, y) \otimes F_{out}(I \mod N_{LIC}, x, y) $$
   Where $\otimes$ is a combination operator (e.g., blending, multiplication) defined in the parent LIC method's section 3.1 [S1, unofficial].

**Edge Cases and Pitfalls:**
- The assumption that $T$ is a multiple of $L$ is required for the step calculation [S1, unofficial].
- Box filters can cause artifacts when boxes reenter the interval; solutions exist in cited literature [S1, unofficial].
- The LIC volume values primarily consist of noise signals for texture construction and do not have physical meaning, requiring specific visualization techniques like volume rendering or isosurface extraction [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Increases with the ratio $T/L$. The dye advection requires $X = N_{LIC} \times (T/L)$ steps, while standard LIC remains at $N_{LIC}$ steps for unaffected cells [S1, unofficial].
- **Memory Complexity:** Requires storing intermediate outputs for both standard LIC and dye advection if combined post-computation, or careful buffering if integrated [S1, unofficial].
- **Strengths:** Allows long dye trails to highlight local flow features without degrading global LIC contrast or slowing down the standard LIC computation [S1, unofficial].
- **Limitations:** Requires careful handling of box filter artifacts. The combination operator $\otimes$ is not detailed in the source text [S1, unofficial].

## Background
This method was introduced in the context of visualizing vector fields using Line Integral Convolution and Dye Advection [S1, unofficial]. It addresses the limitations of standard LIC, which uses a single global convolution length for all cells [S1, unofficial]. The technique is part of a broader effort to enhance the clarity of flow visualization by separating the texture generation (LIC) from the feature highlighting (dye) [S1, unofficial].

## Sources
[S1] NASA Technical Reports Server (NTRS) 19970011183: Visualizing Vector Fields Using Line Integral Convolution and Dye Advection, https://archive.org/details/NASA_NTRS_Archive_19970011183, fetched 2026-10-26 (unofficial)
