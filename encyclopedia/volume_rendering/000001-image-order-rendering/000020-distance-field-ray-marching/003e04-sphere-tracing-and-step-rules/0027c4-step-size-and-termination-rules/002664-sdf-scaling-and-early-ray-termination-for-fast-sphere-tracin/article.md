# SDF Scaling and Early Ray Termination for Fast Sphere Tracing

Also called: SDF Scaling, Two-pass SDF Scaling

## Overview
This method accelerates sphere tracing by using a single low-resolution buffer to provide a "head start" for full-resolution rays, reducing the distance rays must travel to reach surfaces [S1]. It differs from sibling methods like [Standard Constant Step Ray Marching](1080-standard-constant-step-ray-marching.md) by dynamically adjusting ray origins based on pre-computed distance approximations, and from [Inverse Pyramid Rendering](52-inverse-pyramid-rendering.md) by using a single buffer instead of a pyramidal hierarchy, thereby reducing memory cost and complexity [S1].

## How it works
The method operates in two passes: a low-resolution pre-processing pass and a full-resolution main pass.

### Inputs and Outputs
*   **Inputs:** Scene SDFs, camera parameters (origin $\mathbf{o}$, direction $\mathbf{d}$), target resolution $R_{target}$, low-resolution scale factor $s_{low}$ (e.g., 0.25 per axis), and termination epsilon $\epsilon_{low}$ (larger than standard $\epsilon$).
*   **Outputs:** Final rendered image at $R_{target}$.

### Data Structures
*   **Low-Resolution Buffer:** A 2D array of size $W_{low} \times H_{low}$, where $W_{low} = W_{target} / s_{low}$ and $H_{low} = H_{target} / s_{low}$. Each cell stores the approximated distance to the nearest surface along the ray cast from the camera center for that pixel [S1].
*   **Ray Bundle Mapping:** A mapping from full-resolution pixels to low-resolution pixels. Each low-resolution pixel corresponds to a $s_{low} \times s_{low}$ block of full-resolution pixels (e.g., $4 \times 4 = 16$ pixels) [S1].

### Steps

1.  **Pre-processing (Low-Resolution Pass):**
    For each pixel $(u, v)$ in the low-resolution grid:
    a.  Construct a ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}_{uv}$, where $\mathbf{d}_{uv}$ is the direction for pixel $(u, v)$.
    b.  Perform sphere tracing with **SDF Scaling** and **Early Ray Termination**:
        i.  Initialize $t = 0$.
        ii. While $t < t_{max}$:
            1.  Evaluate SDF at $\mathbf{p} = \mathbf{r}(t)$.
            2.  **SDF Scaling Logic:**
                *   If the current SDF value $d(\mathbf{p}) > 0$ (camera/outside): Scale up the SDF volume (inflate). This is typically done by modifying the SDF evaluation function to return a slightly larger distance or by pre-scaling the implicit surface definition.
                *   If $d(\mathbf{p}) < 0$ (inside): Scale down the SDF volume (deflate).
                *   **Hole Handling:** If the SDF represents a hole (via subtraction), decrease its volume size (deflate) if the camera is in front of the implicit surface; otherwise, scale up if the camera is behind [S1].
            3.  **Early Ray Termination:** If the scaled distance $d_{scaled}(\mathbf{p}) < \epsilon_{low}$ (where $\epsilon_{low} > \epsilon_{standard}$), terminate the ray. Store $t$ in the low-resolution buffer at $(u, v)$ [S1].
            4.  Update $t \leftarrow t + d_{scaled}(\mathbf{p})$.
    c.  Store the final $t$ in the low-resolution buffer [S1].

2.  **Main Pass (Full-Resolution):**
    For each pixel $(U, V)$ in the full-resolution grid:
    a.  Determine the corresponding low-resolution pixel $(u, v) = (\lfloor U / s_{low} \rfloor, \lfloor V / s_{low} \rfloor)$.
    b.  Retrieve the approximated distance $t_{approx}$ from the low-resolution buffer at $(u, v)$ [S1].
    c.  **Ray Origin Reconstruction:** Set the new ray origin $\mathbf{o}' = \mathbf{o} + t_{approx} \mathbf{d}_{UV}$. This positions the ray closer to the surface [S1].
    d.  Perform standard sphere tracing from $\mathbf{o}'$ with direction $\mathbf{d}_{UV}$ using the standard epsilon $\epsilon_{standard}$ to refine the intersection and compute shading [S1].
    e.  For shadow rays, use $t_{approx}$ to determine shadow hardness $k$ (higher $k$ yields harder shadows) and trace shadows from $\mathbf{o}'$ [S1].

### Formulas
*   **Ray Update:** $\mathbf{p}_{i+1} = \mathbf{p}_i + \Delta s_i \mathbf{d}$
*   **SDF Scaling Condition:**
    $$
    d_{scaled}(\mathbf{p}) = \begin{cases} 
    \text{Inflate}(d(\mathbf{p})) & \text{if } d(\mathbf{p}) > 0 \\
    \text{Deflate}(d(\mathbf{p})) & \text{if } d(\mathbf{p}) < 0 
    \end{cases}
    $$
    *Note: The exact implementation of Inflate/Deflate is not specified in the sources, but it involves uniformly scaling the SDF volume [S1].*
*   **Termination Condition (Pass 1):**
    $$ \text{Terminate if } d_{scaled}(\mathbf{p}) < \epsilon_{low} $$
*   **Ray Origin Reconstruction:**
    $$ \mathbf{o}' = \mathbf{o} + t_{approx} \mathbf{d} $$

### Parameters
*   **Low-Resolution Scale:** $s_{low} = 0.25$ (buffer is 1/4 resolution per axis) [S1].
*   **Epsilon:** $\epsilon_{low} > \epsilon_{standard}$. The exact value is not given, but it must be large enough to terminate early but small enough to avoid excessive rounding artifacts [S1].
*   **Shadow Hardness $k$:** Adjustable parameter for shadow quality [S1].

### Edge Cases and Pitfalls
*   **Rounding Artifacts:** High $\epsilon_{low}$ values cause shapes with edges to become rounded, eliminating sharp edges and providing an inaccurate approximation [S1].
*   **Surface Overlaps:** SDF scaling may cause surfaces to overlap in the low-resolution pass. Early ray termination with a larger epsilon helps avoid bypassing edges of smaller SDFs [S1].
*   **Hole Approximation:** Incorrect scaling for holes (e.g., scaling up when the camera is behind) can lead to inaccurate distance estimates [S1].

## Operating characteristics
*   **Performance:** Achieves a 3x speedup in frame time compared to state-of-the-art methods without anti-aliasing on an NVIDIA RTX-3070Ti at 4K resolution in the 'Primitives' scene [S1].
*   **Shadow Ray Performance:** Shadow rays often finish after 1 step, achieving a 2.1x improvement over ground truth in the 'Primitives' scene, 1.52x in 'Columns', 1.8x in 'Greek Temple', and 1.5x in 'Sponza atrium' [S1].
*   **Memory:** Uses a single low-resolution buffer, reducing memory cost compared to pyramidal approaches [S1].
*   **Limitations:** Visual quality may suffer from rounded edges if $\epsilon_{low}$ is too high. Performance gains diminish in highly complex scenes with many SDFs (e.g., 'Sponza atrium') due to increased computation for SDF evaluation [S1].

## Background
The method was introduced in the paper "Skipping Spheres: SDF Scaling & Early Ray Termination for Fast Sphere Tracing" [S1]. It draws inspiration from prior techniques aimed at reducing ray steps by adjusting step size and generating lower resolution renders [PKM23]. It is designed to minimize artifacts while significantly cutting down steps compared to previous methods like Inverse Pyramid Rendering [S1].

## Sources
[S1] Skipping Spheres: SDF Scaling & Early Ray Termination for Fast Sphere ..., https://diglib.eg.org/items/7f21e58c-639e-4359-bf94-414738612006, fetched 2026-10-27
