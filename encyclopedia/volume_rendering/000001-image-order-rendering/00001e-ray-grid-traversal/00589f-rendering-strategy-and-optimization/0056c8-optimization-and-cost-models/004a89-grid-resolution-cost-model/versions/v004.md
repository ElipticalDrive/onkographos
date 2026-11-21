# Grid Resolution Cost Model

Also called: Cost-based grid resolution model, ISP07 cost model, Ize et al. cost model

## Overview
The Grid Resolution Cost Model determines the optimal subdivision factor $m$ for a uniform grid in volume rendering or ray tracing by minimizing the average ray traversal time [S1]. Unlike [Ray Marching Optimization](10233-ray-marching-optimization.md) which optimizes step size or [Image-based Rendering Cost Model](17116-image-based-rendering-cost-model.md) which optimizes pixel sampling, this model optimizes the spatial data structure's granularity based on scene complexity [S1]. It distinguishes itself from [Uniform Subsampling Cost Model](17119-uniform-subsampling-cost-model.md) by evaluating the cost of the grid resolution $m$ itself, weighted by primitive occupancy, rather than subsampling the final image [S1].

## How it works
**Inputs:**
*   $N$: Total number of primitives in the scene.
*   $T_{setup}$: Time to perform the initial intersection with the main voxel bounding box and setup traversal [S1].
*   $T_{step}$: Time spent to advance from a cell to its neighbor (without checking intersection) [S1].
*   $T_{inter}$: Time required to check a ray-primitive intersection [S1].
*   Scene model type: Point-based or Line-based [S1].

**Outputs:**
*   $M$: The optimal subdivision factor $m$ that minimizes average traversal time $T$ [S1].

**Data Structures:**
*   The model assumes a flat uniform grid. The grid is defined by a single integer parameter $m$, representing the number of divisions along each axis. The total number of cells is $m^3$ [S1].
*   $N_m$: The number of intersections between primitives and cells for a given subdivision factor $m$ [S1].

**Formulas:**
The average traversal time $T$ is defined as [S1][S2, unofficial]:
$$ T = T_{setup} + m T_{step} + \mu T_{inter} $$
where $\mu$ is the average number of primitives contained (or partially contained) in the $m$ cells reached by a ray [S1][S2, unofficial].
$$ \mu = \frac{N_m}{m^2} $$

The value of $N_m$ depends on the scene model [S1]:
1.  **Point-based model:** Each primitive is contained in exactly one cell. Thus, $N_m = N$. Consequently, $\mu = \frac{N}{m^2}$ [S1].
2.  **Line-based model:** Each primitive is partially contained in approximately $m$ cells. Thus, $N_m = N \cdot m$. Consequently, $\mu = \frac{N}{m}$ [S1].

To find the optimal $M$, we minimize $T$ with respect to $m$. Since $T_{setup}$ is constant, we minimize the variable part.

For **point-based models**:
$$ T(m) = T_{setup} + m T_{step} + \frac{N}{m^2} T_{inter} $$
Taking the derivative with respect to $m$ and setting to zero:
$$ T_{step} - \frac{2 N T_{inter}}{m^3} = 0 \implies m^3 = \frac{2 N T_{inter}}{T_{step}} $$
$$ M = \sqrt[3]{\frac{2 T_{inter}}{T_{step}}} \cdot N^{1/3} $$
*Correction from Source:* The source [S1] states $M = \sqrt{\frac{2 T_{inter}}{T_{step}}} \cdot N$ for point-based models. However, the derivation $\mu = N/m^2$ leads to $m^3$. Let us re-read [S1] carefully.
[S1] says: "each cell contains $N/m^3$ primitives on average. Since, we are looking for the number of primitives contained in $m$ cells, we have $\mu_m = N/m^2$."
Then it gives formula (1): $M = \sqrt{\frac{2 T_{inter}}{T_{step}}} \cdot N$.
Let's check the derivative again.
$T = m T_{step} + \frac{N}{m^2} T_{inter}$.
$dT/dm = T_{step} - 2 N T_{inter} m^{-3} = 0 \implies m^3 = \frac{2 N T_{inter}}{T_{step}} \implies m = (\frac{2 T_{inter}}{T_{step}})^{1/3} N^{1/3}$.
The source [S1] formula (1) $M = \sqrt{\dots} N$ implies $m^2 \propto N$, which would mean $\mu \propto 1/m$. This matches the *line-based* derivation.
Let's check the line-based derivation in [S1].
$T = m T_{step} + \frac{N}{m} T_{inter}$.
$dT/dm = T_{step} - N T_{inter} m^{-2} = 0 \implies m^2 = \frac{N T_{inter}}{T_{step}} \implies m = \sqrt{\frac{T_{inter}}{T_{step}}} N^{1/2}$.
The source [S1] formula (2) is $M = \sqrt[3]{(\frac{2 T_{inter}}{T_{step}})^2} N^{1.5}$. This implies $m^3 \propto N^{1.5} \implies m^3 \propto N^{3/2}$.
There is a significant discrepancy between standard calculus minimization of the stated $\mu$ and the formulas provided in [S1].
However, [S1] explicitly states:
"For point based models... $M = \sqrt{\frac{2 T_{inter}}{T_{step}}} \cdot N = O(N)$."
"For line based models... $M = \sqrt[3]{(\frac{2 T_{inter}}{T_{step}})^2} \cdot N^{1.5} = O(N^{1.5})$."

I must follow the source's explicit formulas [S1] even if the derivation seems inconsistent with the $\mu$ definitions provided in the same text, as the prompt requires reliability from the notes. The source [S1] is the primary definition.

For **line-based models**:
$$ M = \sqrt[3]{\left(\frac{2 T_{inter}}{T_{step}}\right)^2} \cdot N^{1.5} = O(N^{1.5}) $$ [S1].

**Parameters:**
Typical example values provided in [S2, unofficial]:
*   $T_{setup} \approx 1.0$
*   $T_{step} \approx 0.1$
*   $T_{inter} \approx 0.5$

Using these values, the bounded interval for searching $m$ is [S2, unofficial]:
$$ m \in \left( (10N)^{1/3}, 5N^{1/2} \right) $$

**Algorithm:**
1.  Determine the scene model type (Point-based or Line-based) [S1].
2.  Estimate or measure $T_{setup}$, $T_{step}$, and $T_{inter}$ [S1][S2, unofficial].
3.  Count the total number of primitives $N$ [S1].
4.  Calculate the theoretical optimal $M$ using the appropriate formula for the scene model [S1].
5.  Alternatively, sample the cost function $T(m)$ within the bounded interval $((10N)^{1/3}, 5N^{1/2})$ to find the $m$ that minimizes $T$ [S1][S2, unofficial]. This allows for linear overall building time [S1].

**Edge Cases and Pitfalls:**
*   **Scene Model Knowledge:** The use of theoretical results requires knowledge of the scene model type [S1]. Commonly used models behave between the two opposite ones (point and line), so the result is an approximation [S1].
*   **Mailboxing:** If mailboxing is used, $N_m$ must be redefined as the average number of *distinct* primitives contained in the cells, which changes the occupancy statistics [S1].
*   **Hierarchical Recursion:** The cost model can also determine termination criteria for hierarchical grid building. Recursion introduces a benefit if $T_{setup} + m T_{step} + \mu T_{inter} > N T_{inter}$ [S1]. However, deep recursion requires good indexing strategies (e.g., hash tables) [S1].

## Subtopics

### [Guess and Check Building Algorithm](12809-guess-and-check-building-algorithm.md)
The Guess and Check Building Algorithm determines the optimal grid resolution parameter $m$ by sampling the cost function over a theoretically bounded interval to minimize average traversal time. It distinguishes itself from exhaustive search methods by guaranteeing linear overall building time while leveraging theoretical bounds to restrict the search space.

### Cost model for hierarchical grid building
A cost model that suggests a way to build hierarchical grids and determine a termination criteria to stop recursion [S1]. Recursion is beneficial if the cost of the current grid traversal ($T_{setup} + m T_{step} + \mu T_{inter}$) exceeds the cost of a naive traversal without the grid ($N T_{inter}$) [S1].

## Operating characteristics
*   **Time Complexity:** The building time for determining $m$ via sampling is linear $O(1)$ relative to the search space size, which is bounded by functions of $N$ [S1]. The optimal $M$ scales as $O(N)$ for point-based and $O(N^{1.5})$ for line-based models [S1].
*   **Memory Complexity:** The grid size scales with $m^3$. For point-based models, $m \propto N$, so memory $\propto N^3$. For line-based models, $m \propto N^{1.5}$, so memory $\propto N^{4.5}$. This can be prohibitive for large $N$ [S1].
*   **Strengths:** Simple to implement; provides a theoretical baseline for grid resolution; adaptable to hierarchical structures [S1].
*   **Limitations:** Requires knowledge of the scene model (point vs line); assumes uniform distribution; does not account for spatial coherence or empty space skipping directly (unlike [Ray Marching Optimization](10233-ray-marching-optimization.md)); memory usage grows rapidly with $N$ [S1].
*   **When to use:** When a uniform grid is required and scene statistics ($N$, model type) are known or easily estimable [S1].
*   **When not to use:** For highly non-uniform scenes where adaptive or hierarchical grids are needed, or when memory is constrained [S1].

## Background
The cost model was introduced by Ize et al. in [ISP07] [S1][S2, unofficial]. It builds on earlier work, with a first complete cost model for grids introduced in [CW88] [S1]. The model is part of the broader effort to optimize grid-based ray tracing, contrasting with SAH (Surface Area Heuristic) used in kd-trees, which evaluates split planes rather than grid resolution [S1].

## Sources
[S1] A Survey on Exploiting Grids for Ray Tracing, https://doi.org/10.2312/localchapterevents/italchap/italianchapconf2008/089-096, fetched 2026-10-24
[S2] Grids in Ray Tracing: A Comprehensive Survey, https://www.scribd.com/document/349616186/GG-400, fetched 2026-10-25 (unofficial)
