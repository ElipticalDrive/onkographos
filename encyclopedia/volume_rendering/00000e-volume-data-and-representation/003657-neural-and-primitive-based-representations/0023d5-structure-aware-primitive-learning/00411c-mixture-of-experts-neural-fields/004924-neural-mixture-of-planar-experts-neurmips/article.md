# Neural Mixture of Planar Experts (NeurMiPs)
## Overview
Neural Mixture of Planar Experts (NeurMiPs) is a scene representation that models geometry and appearance using a collection of local planar experts, each defined by an oriented 2D rectangle and a neural radiance field [S1]. It distinguishes itself from volume-based methods by being sample-efficient and reflecting true surface geometry, while offering higher rendering speed and lower memory usage than neural surface rendering [S1].
## How it works
**Inputs and Outputs:**
*   **Input:** A set of input images with known camera poses [S1].
*   **Output:** Novel view images synthesized via ray casting and alpha blending [S1].

**Data Structures:**
The scene is represented as a mixture of $K$ planar experts [S1]. Each expert $k$ consists of:
1.  **Geometry Parameters ($s_k$):**
    *   $p_k \in \mathbb{R}^3$: The 3D center of the rectangle [S1].
    *   $n_k \in \mathbb{R}^3$: The normalized plane normal [S1].
    *   $u_k \in \mathbb{R}^3$: The normalized up vector [S1].
    *   $w_k \in \mathbb{R}$: The width of the rectangle [S1].
    *   $h_k \in \mathbb{R}$: The height of the rectangle [S1].
2.  **Appearance Model:** A neural radiance field function $f_k: \mathbb{R}^3 \times \mathbb{R}^3 \to \mathbb{R}^4$ that takes a 3D coordinate $x$ and ray direction $d$ as input, outputting color $c \in \mathbb{R}^3$ and opacity $\alpha \in \mathbb{R}$ [S1].

**Numbered Steps:**
1.  **Initialization:** Not given in the sources.
2.  **Training:** Learn both the geometry parameters and the radiance field weights end-to-end from the input images [S1].
3.  **Ray Casting:** For each pixel in the target image, cast a ray $r(t) = o + td$ from the camera origin $o$ with direction $d$ [S1].
4.  **Intersection Check:** Conduct a ray-rectangle intersection check to identify intersecting points and planes [S1].
5.  **Point Evaluation:** Evaluate the color and transparency at each intersecting point based on the point's coordinate using the plane's neural radiance field [S1].
6.  **Compositing:** Calculate the final ray color through alpha blending the colors of all intersecting points along the ray [S1].

**Formulas:**
*   **Ray-Plane Intersection:** For a plane defined by center $p_k$ and normal $n_k$, the intersection distance $t$ for ray $r(t) = o + td$ is found by solving $(o + td - p_k) \cdot n_k = 0$, yielding $t = \frac{(p_k - o) \cdot n_k}{d \cdot n_k}$ [G]. If $d \cdot n_k \approx 0$, the ray is parallel to the plane and does not intersect [G].
*   **Point-in-Rectangle Check:** Let the intersection point be $x$. Define the local right basis vector $v_k = n_k \times u_k$ [G]. The point lies inside the rectangle if $|(x - p_k) \cdot v_k| \le \frac{w_k}{2}$ and $|(x - p_k) \cdot u_k| \le \frac{h_k}{2}$ [G].
*   **Alpha Blending:** The final color $C$ is computed as $C = \sum_{i} c_i \alpha_i \prod_{j < i} (1 - \alpha_j)$, where $i$ indexes the sorted intersecting points along the ray [S1].

**Parameters:**
*   $K$: Number of planar experts. Typical values depend on scene complexity; for example, 500 planes can fit a $5 \times 5$ m² scene with a maximum point-to-surface error of 8.66 mm [S1].

**Edge Cases and Pitfalls:**
*   **Parallel Rays:** If $d \cdot n_k \approx 0$, the ray is parallel to the plane and does not intersect [G].
*   **Degenerate Planes:** Not given in the sources.
*   **Precision:** Floating-point precision in intersection tests may cause gaps or overlaps between adjacent planes [G].
## Operating characteristics
*   **Time Complexity:** Rendering time depends on the number of planes $K$ and the number of intersections per ray, as each ray only hits a small subset of surfaces [S1].
*   **Memory Complexity:** Memory usage is proportional to $K$ times the size of the neural network parameters for each expert, making it more memory-efficient than dense volume representations [S1].
*   **Performance:** Compared to neural surface rendering, the method is efficient in both memory and computation [S1]. Compared to volume rendering, it is significantly more sample-efficient with better extrapolation [S1]. Compared to multi-plane imagery, it better reflects the geometry [S1].
*   **Strengths:** Efficient in memory and computation; sample-efficient; better extrapolation than volume rendering; reflects geometry better than multi-plane imagery [S1].
*   **Limitations:** Not given in the sources.
*   **When to Use:** Man-made environments with locally planar surfaces [S1].
*   **When Not to Use:** Not given in the sources.
## Background
NeurMiPs was proposed by Zhihao Lin et al. [S1]. The concept of modeling real-world surfaces through piece-wise local planar structures is not new, having been used in vision and graphics research [S1]. The method blends the efficiency of explicit mesh rendering with the flexibility of neural radiance fields [S1].
## Sources
[S1] NeurMiPs: Neural Mixture of Planar Experts for View Synthesis, https://doi.org/10.1109/cvpr52688.2022.01525, fetched 2026-10-26
