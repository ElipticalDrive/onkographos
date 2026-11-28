# Variable Convolution Length for Dye
Also called: Separate dye convolution length

## Overview
Variable Convolution Length for Dye decouples the convolution length of the dye advection from the global Line Integral Convolution (LIC) texture [S1, unofficial]. It solves the problem where extending the global convolution length to highlight long-range flow features degrades the contrast of the LIC texture and increases computational cost [S1, unofficial]. By allowing a separate, longer convolution length for the dye, it enables long dye trails without compromising the efficiency or visual quality of the underlying LIC [S1, unofficial].

## How it works
**Inputs:** Vector field defining the flow, scalar noise field for LIC texture, global convolution length $L$, dye convolution length $T$, and standard LIC animation steps $steps$ [S1, unofficial].
**Outputs:** Final composite scalar field combining LIC texture and dye advection [S1, unofficial].
**Data Structures:** Not given in the sources.
**Numbered Steps:**
1. Calculate the number of animation steps required for the dye advection [S1, unofficial].
2. For cells not affected by the dye, compute only $steps$ steps of standard LIC output [S1, unofficial].
3. For dyed cells, compute the dye advection output $D_{out}(I,x,y)$ at animation step $I \in [0, X-1]$ using a box filter along the streamline [S1, unofficial].
4. Combine the standard LIC output with the dye advection output [S1, unofficial].
**Formulas:**
- Number of animation steps for dye: $X = steps \times \frac{T}{L}$ [S1, unofficial].
- LIC formula for dyed advection: $D_{out}(I,x,y) = \frac{\sum_{i=0}^{Z-1} D_{in}(P_i) \cdot \mathbb{I}(S_i)}{\sum_{i=0}^{Z-1} \mathbb{I}(S_i)}$ [S1, unofficial].
- Combine standard LIC and dye advection: $Final_{out}(I, x, y) = D_{out}(I, x, y) \otimes F_{out}(I \mod steps, x, y)$ [S1, unofficial].
**Parameters:** $L$ (global convolution length), $T$ (dye convolution length), $steps$ (standard LIC steps), $Z$ (number of samples in the filter), $\Delta L$ (length of filter box), $r$ (streamline position) [S1, unofficial]. Assumption: $T$ is a multiple of $L$ [S1, unofficial].
**Edge Cases and Pitfalls:**
- Requires $T$ to be a multiple of $L$ for the step calculation [S1, unofficial].
- Box filters can cause artifacts when boxes reenter the interval; solutions exist in cited literature [S1, unofficial].
- The LIC volume values consist primarily of noise signals for texture construction and lack physical meaning, requiring specific visualization techniques like volume rendering or isosurface extraction [S1, unofficial].
- The exact definition of the combination operator $\otimes$ is not detailed in the source text [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Increases with the ratio $T/L$ [S1, unofficial]. The dye advection requires $X = steps \times (T/L)$ steps, while standard LIC remains at $steps$ steps for unaffected cells [S1, unofficial].
- **Memory Complexity:** Not given in the sources.
- **Strengths:** Allows long dye trails to highlight local flow features without degrading global LIC contrast or slowing down the standard LIC computation [S1, unofficial].
- **Limitations:** Requires careful handling of box filter artifacts [S1, unofficial]. The combination operator is not detailed in the source text [S1, unofficial].

## Background
This method was introduced in the context of visualizing vector fields using Line Integral Convolution and Dye Advection [S1, unofficial]. It addresses the limitations of standard LIC, which uses a single global convolution length for all cells [S1, unofficial]. The technique is part of a broader effort to enhance the clarity of flow visualization by separating the texture generation (LIC) from the feature highlighting (dye) [S1, unofficial].

## Sources
[S1] NASA Technical Reports Server (NTRS) 19970011183: Visualizing Vector Fields Using Line Integral Convolution and Dye Advection, https://archive.org/details/NASA_NTRS_Archive_19970011183, fetched 2026-10-26 (unofficial)
