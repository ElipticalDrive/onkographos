# Haloed Line Integral Convolution

Also called: Depth Discontinuity Halos, Haloed LIC, Haloed Line Effect for 3D Flow, LIC with visibility-impeding halos, Visibility-Impeding Halos

## Overview
Haloed Line Integral Convolution is a volume rendering technique that uses an asymmetric filter kernel and halo detection to emphasize depth discontinuities between overlapping streamlines, preventing visibility-impeding artifacts [S1, unofficial]. It builds on [Line Integral Convolution (LIC)](7940-line-integral-convolution-lic.md) by generating two simultaneous textures (streamlines and halos) and modifying the raycasting accumulation to boost opacity when exiting a halo region, thereby highlighting the depth extent of 3D flow structures [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A 3D vector field (for streamline tracing), a scalar noise field (for LIC convolution), and a scalar field defining the "halo" width (often derived from the same noise or a separate mask). The implementation assumes a black background [S1, unofficial].
*   **Outputs:** A rendered 2D image where overlapping streamlines are separated by dark gaps (halos) indicating depth discontinuities.

**Data Structures:**
1.  **Streamline Texture ($T_{stream}$):** A 3D volume storing the integrated LIC values for the primary streamlines.
2.  **Halo Texture ($T_{halo}$):** A 3D volume storing the integrated LIC values for the halo regions. This is generated simultaneously with $T_{stream}$ using concentric input spots.
3.  **Filter Array ($v$):** An array of fixed bounds used for incremental LIC calculation. At each step, the current origin is denoted $v_0$, and the intensity of the next point is denoted $I_{+1}$ or $I_{-1}$ depending on the direction of travel [S1, unofficial].

**Parameters:**
*   $c$: A constant controlling the rate of decrease in contribution for samples farther along the streamline. The sources specify $c < 1$ [S1, unofficial].
*   Background Color: Black [S1, unofficial].

**Algorithm Steps:**

1.  **Generate Textures:** Perform Line Integral Convolution simultaneously over two input textures containing identically located spots of concentric sizes to generate the streamline texture ($T_{stream}$) and the halo texture ($T_{halo}$). This ensures streamline tracing is done only once for the pair, minimizing overhead [S1, unofficial].

2.  **Raycasting Initialization:** For each pixel, initialize accumulated color $C_{acc} = 0$ and accumulated opacity $\alpha_{acc} = 0$. Trace rays through both $T_{stream}$ and $T_{halo}$ volumes simultaneously [S1, unofficial].

3.  **Accumulation Loop:** For each sample point along the ray:
    *   Read the streamline value $v_{stream}$ and halo value $v_{halo}$ at the current position.
    *   Update the streamline LIC value using the asymmetric filter kernel equations (see Formulas).
    *   Accumulate color and opacity from $T_{stream}$ into $C_{acc}$ and $\alpha_{acc}$ using standard volume rendering compositing:
        $$ C_{acc} = C_{acc} + v_{stream} \cdot (1 - \alpha_{acc}) $$
        $$ \alpha_{acc} = \alpha_{acc} + v_{stream} \cdot (1 - \alpha_{acc}) $$
    *   Check if the ray has exited a halo region. A "halo region" is defined by the non-zero values in $T_{halo}$.

4.  **Halo Exit Handling:** Upon exiting a halo (transitioning from $v_{halo} > 0$ to $v_{halo} = 0$):
    *   Let $\alpha_{max}$ be the maximum density (opacity) encountered in the previously traversed halo region.
    *   Increase the accumulated opacity by an amount proportional to $\alpha_{max}$. The source states the opacity is increased by an amount proportional to the maximum density encountered in the previously traversed halo region [S1, unofficial].
    *   *Note:* The exact proportional constant is not specified in the sources. [G] A common implementation would add $\alpha_{max}$ directly or scale it by a visibility factor. Given the goal is to create a "gap" or "darkness," this step effectively darkens the background or reduces the contribution of subsequent layers. However, the text says "increase the accumulated opacity," which in standard compositing makes the pixel more opaque. To create a "gap" (darkness) against a black background, increasing opacity of a dark color is the mechanism. If the background is black, increasing opacity of the *halo* value (which might be dark) or simply blocking the view is the key. The text says "depth discontinuities are highlighted by gaps." The mechanism described is: "increase the accumulated opacity by an amount proportional to the maximum density encountered in the previously traversed halo region." This implies the halo region itself contributes to the opacity. When you exit, you have already accumulated the halo's color/opacity. The "boost" might refer to ensuring the halo's presence is felt. Let's stick strictly to the text: "increase the accumulated opacity by an amount proportional to the maximum density encountered in the previously traversed halo region."

5.  **Normal Rendering:** Render voxels lying between the entrance and exit points of the first-encountered halo in the normal fashion (without the halo-induced opacity boost). This ensures the interior of the streamline is rendered correctly [S1, unofficial].

**Formulas:**
The incremental calculation for the LIC texture uses an asymmetric filter kernel. Let $I'_0$ be the current integrated output texture value, $v_0$ be the current sample value (origin), and $v_{flength}$ be the value at the end of the filter window. Let $flength$ be the length of the filter window.

For forward integration ($+1$):
$$ I'_{+1} = \frac{I'_0 - v_0}{c} + \frac{v_{flength+1}}{c^{flength}} $$

For backward integration ($-1$):
$$ I'_{-1} = \left( I'_0 - \frac{v_{flength}}{c^{flength}} \right) c + v_{-1} $$

Where:
*   $I'_0$: Current integrated value.
*   $v_0$: Current input sample value.
*   $v_{flength}$: Input sample value at the end of the filter window.
*   $v_{flength+1}$: Next input sample value (for forward).
*   $v_{-1}$: Previous input sample value (for backward).
*   $c$: Constant $< 1$ controlling the decay rate [S1, unofficial].

**Pitfalls:**
*   If halos of different lines overlap in 3-space but the lines themselves do not intersect, the method will not indicate the existence of depth discontinuities between those lines [S1, unofficial].
*   The method assumes a black background.

## Operating characteristics
*   **Time Complexity:** The texture generation is efficient because streamline tracing is performed only once for the pair of volumes (streamline and halo) [S1, unofficial]. The raycasting step involves simultaneous traversal of two volumes, adding minimal overhead compared to single-volume LIC.
*   **Memory Complexity:** Requires storage for two 3D textures ($T_{stream}$ and $T_{halo}$) plus the filter array for incremental calculation.
*   **Strengths:** Efficiently computes 3D flow textures that reveal directional information using an asymmetric filter kernel in combination with fast-LIC [S1, unofficial]. Effectively conveys depth extent and discontinuities in overlapping flow structures [S1, unofficial].
*   **Limitations:** Cannot distinguish depth discontinuities between non-intersecting lines if their halos overlap [S1, unofficial]. Limited to steady flow data in the described implementation (unsteady flow is mentioned as future work) [S1, unofficial].

## Background
This method was developed by Wegkittl et al. and published in IEEE Computer Graphics and Applications in July/August 1998 [S1, unofficial]. It was supported by ICASE under NASA contract NAS1-19480. The work builds upon previous LIC techniques and aims to improve the visualization of 3D flow data by addressing the ambiguity of overlapping streamlines. The authors acknowledge Marc Levoy for providing the original volume rendering platform [S1, unofficial].

## Sources
[S1] Visualizing 3D Flow, https://www-users.cse.umn.edu/~interran/papers/cga98.pdf, fetched 2026-10-29 (unofficial)
