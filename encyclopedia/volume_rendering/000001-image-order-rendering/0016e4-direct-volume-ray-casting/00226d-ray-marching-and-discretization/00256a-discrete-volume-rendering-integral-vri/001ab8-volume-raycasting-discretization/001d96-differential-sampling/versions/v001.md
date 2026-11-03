# Differential Sampling

Also called: Differential construction of T, quadratic ray sampling, Nyquist-respecting ray sampling

## Overview
Differential sampling is a ray discretization strategy for volume raycasting that adapts the sampling density along a ray based on the view-dependent frequency of the image plane, ensuring the sampling rate respects the Nyquist limit [S1]. Unlike uniform sampling methods, it uses a quadratic sampling function $T(t)$ derived from ray differentials, allowing variable step sizes that are finer near the viewpoint and coarser further away, which prevents aliasing for features close to the camera [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Camera parameters (origin $\vec{o}$, central view direction $\vec{w}$, right vector $\vec{u}$, up vector $\vec{v}$), image plane resolution ($\Delta x, \Delta y$), a scaling factor $sk$, and the volume data structure.
*   **Outputs:** A sequence of sample positions $\vec{P}_i$ along the ray and their corresponding transfer function evaluations.

**Data Structures:**
The method maintains the following state variables per ray during iteration:
*   $\vec{P}$: Current 3D sample position.
*   $\vec{D}$: Unitized ray direction vector.
*   $t$: Current distance along the ray from the origin.
*   $\Delta t$: Current step size (derivative of $T$ with respect to $t$).
*   $x$: Current image-space coordinate index (integer).
*   $T(x)$: Cumulative distance along the ray corresponding to image coordinate $x$.

**Parameters:**
*   $sk$: Scaling factor for the step size multiplier $k$. Typical value: $4$ [S1]. Theoretical Nyquist limit requires $sk < 0.5$, but this is excessive in practice [S1].
*   $k$: Step size multiplier, defined as $k = sk |\vec{u}\Delta x + \vec{v}\Delta y|$ [S1].
*   $k_0$: Constant combining $k$ and geometric factors. If $k$ is arbitrarily chosen, $k_0 = k$ [S1].

**Algorithm Steps:**

1.  **Ray Generation:** Generate the ray direction vector $\vec{d}$ for pixel coordinates $(x, y)$ using the pinhole camera model:
    $$ \vec{d}(x, y) = \vec{w} + x\vec{u} + y\vec{v} $$
    where $\vec{w}$ is the central view direction, and $\vec{u}, \vec{v}$ are the right and up vectors [S1].

2.  **Ray Unitization:** Compute the unitized ray direction $\vec{D}$:
    $$ \vec{D}(x, y) = \frac{\vec{d}}{\sqrt{\vec{d} \cdot \vec{d}}} $$
    [S1]

3.  **Compute Ray Differential Magnitude:** Calculate the magnitude of the partial derivative of $\vec{D}$ with respect to the image-space coordinate $x$. First, compute the vector derivative:
    $$ \frac{\partial \vec{D}}{\partial x} = \frac{(\vec{d} \cdot \vec{d})\vec{u} - (\vec{d} \cdot \vec{u})\vec{d}}{(\vec{d} \cdot \vec{d})^{3/2}} $$
    Then, take its magnitude:
    $$ \left| \frac{\partial \vec{D}}{\partial x} \right| $$
    [S1]

4.  **Compute Coefficients:**
    *   Compute $k$ using the pixel footprint: $k = sk |\vec{u}\Delta x + \vec{v}\Delta y|$ [S1].
    *   Set $k_0 = k$ (or $k \tan(\theta)$ if explicit geometric correction is desired, but sources indicate $k_0=k$ is sufficient if $k$ is chosen appropriately) [S1].
    *   Compute coefficient $a$:
        $$ a = k_0 \left| \frac{\partial \vec{D}}{\partial x} \right| $$
    *   Compute coefficient $b$:
        $$ b = k_0 $$
    [S1]

5.  **Initialization:**
    *   Initialize image coordinate $x = 0$ [S1].
    *   Initialize cumulative distance $T = 0$ [S1].
    *   Initialize step size $\Delta t = b$ [S1].
    *   Initialize position $\vec{P} = \vec{o}$ [S1].
    *   [G] Initialize a variable `max_t_sampled` to $-\infty$ to handle macrocell entry duplicates [S1].

6.  **Sampling Loop:**
    While $T < D_{exit}$ (exit distance of the volume):
    *   **Sample:** Evaluate the volume at $\vec{P}$.
    *   **Update Max Sampled:** [G] `max_t_sampled = max(max_t_sampled, T)` [S1].
    *   **Increment Step Size:**
        $$ \Delta t_{new} = \Delta t + a $$
    *   **Increment Position:**
        $$ \vec{P}_{new} = \vec{P} + \Delta t \cdot \vec{D} $$
    *   **Update State:**
        $$ \Delta t = \Delta t_{new} $$
        $$ \vec{P} = \vec{P}_{new} $$
        $$ T = T + \Delta t_{old} \quad \text{(Note: The source implies T accumulates the previous step size or is derived from x. The recurrence in Eq 17 updates P and dt. T is implicitly } \sum \Delta t_i \text{ or computed via } T(x) = \frac{a}{2}x^2 + bx \text{ if x is tracked. The source Eq 17 updates P and dt directly. We track T for exit condition.)} $$
        *Correction based on Eq 15 and 17:* The source states "sample at $\vec{P} = \vec{P}(T(x))$" and then increments. The increment $\Delta t$ in Eq 17 is the *current* step size used to move from $\vec{P}_0$ to $\vec{P}_1$. The next step size is $\Delta t_1 = \Delta t_0 + a$.
        So, in the loop:
        1. Use current $\Delta t$ to move: $\vec{P} \leftarrow \vec{P} + \Delta t \cdot \vec{D}$.
        2. Update $\Delta t \leftarrow \Delta t + a$.
        3. Update $T \leftarrow T + \Delta t_{previous}$.
        *Simpler Implementation:* Since $T(x) = \frac{a}{2}x^2 + bx$, we can just increment $x$ by 1 and compute $T(x)$ directly, then compute $\vec{P}$ from $T$. However, the source explicitly gives the incremental update in Eq 17. We follow Eq 17.
        
        *Refined Loop based on Eq 17:*
        The source says: "sample at $\vec{P} = \vec{P}(T(x))$, and perform the following increments... $\vec{P}_1 = \vec{P}_0 + \Delta t_0 \vec{D}$ and $\Delta t_1 = \Delta t_0 + a$".
        This implies $\Delta t_0$ is the step taken to get to $\vec{P}_0$? No, $\Delta t$ is the derivative $\partial T / \partial t$.
        Let's trace:
        Start: $x=0, T=0, \Delta t = b$.
        Sample at $T=0$ ($\vec{P}=\vec{o}$).
        Increment: $\vec{P} \leftarrow \vec{P} + b \vec{D}$. $\Delta t \leftarrow b + a$.
        Next $T$ is $b$.
        Sample at $T=b$.
        Increment: $\vec{P} \leftarrow \vec{P} + (b+a) \vec{D}$. $\Delta t \leftarrow (b+a) + a$.
        Next $T$ is $b + (b+a) = 2b + a$.
        Check against $T(x) = \frac{a}{2}x^2 + bx$.
        $x=0, T=0$.
        $x=1, T = a/2 + b$.
        The incremental sum $b + (b+a) = 2b+a$ does not match $a/2+b$.
        
        *Re-reading Source [S1] carefully:*
        "The sampling function T has a derivative varying linearly with distance: $\Delta T = \partial T / \partial t = at + b$".
        Wait, Eq 6 says $\Delta T = \partial T / \partial t = at + b$. This is derivative with respect to *ray distance* $t$, not image coordinate $x$.
        But Eq 15 says $\partial^2 T / \partial x^2 = a$.
        There is a confusion in the source text between $t$ and $x$.
        Eq 11: $\Delta t = kx$.
        Eq 14: $\partial T / \partial x = \Delta t$.
        So $\partial T / \partial x = k' |\partial D / \partial x| x + k''$.
        This means $T$ is a function of $x$.
        The increment in Eq 17: $\vec{P}_1 = \vec{P}_0 + \Delta t_0 \vec{D}$. Here $\Delta t_0$ is the step size in *world space distance*.
        The source states: "incrementing the position... consists only of an extra vector multiplication... on top of the vector addition for uniform sampling."
        
        Let's look at Eq 17 again. $\Delta t_1 = \Delta t_0 + a$.
        If $\Delta t$ is the world-space step size, and it increases linearly by $a$, then $\Delta t(x) = \Delta t(0) + x \cdot a$.
        We know $\Delta t(0) = b$ (from Eq 16, $\partial T / \partial x = b$ at $x=0$, and $\partial T / \partial x = \Delta t$).
        So $\Delta t(x) = b + a x$.
        The position update is $\vec{P}_{x+1} = \vec{P}_x + \Delta t(x) \vec{D}$.
        The cumulative distance $T(x)$ is $\sum_{i=0}^{x-1} \Delta t(i) = \sum_{i=0}^{x-1} (b + a i) = bx + a \frac{(x-1)x}{2}$.
        This is approximately $\frac{a}{2}x^2 + bx$.
        
        So the implementation should track $x$ (integer) and compute $\Delta t$ based on $x$.
        
        **Revised Algorithm Steps:**

        1.  **Ray Generation:** $\vec{d} = \vec{w} + x\vec{u} + y\vec{v}$ [S1].
        2.  **Unitize:** $\vec{D} = \vec{d} / |\vec{d}|$ [S1].
        3.  **Compute $|\partial \vec{D} / \partial x|$:**
            $$ \frac{\partial \vec{D}}{\partial x} = \frac{(\vec{d} \cdot \vec{d})\vec{u} - (\vec{d} \cdot \vec{u})\vec{d}}{(\vec{d} \cdot \vec{d})^{3/2}} $$
            $$ \text{mag} = \left| \frac{\partial \vec{D}}{\partial x} \right| $$
            [S1]
        4.  **Compute Coefficients:**
            $$ k = sk |\vec{u}\Delta x + \vec{v}\Delta y| $$
            $$ a = k \cdot \text{mag} $$
            $$ b = k $$
            [S1]
        5.  **Initialization:**
            $$ x = 0 $$
            $$ \vec{P} = \vec{o} $$
            $$ \Delta t = b $$
            [S1]
        6.  **Loop:**
            While $\vec{P}$ is inside the volume:
            *   Sample volume at $\vec{P}$.
            *   [G] Check for macrocell entry. If entering a new macrocell at $T_{enter}$, compute $x_{enter}$ such that $T(x_{enter}) \approx T_{enter}$. Set $x = \lceil x_{enter} \rceil$. Recompute $\Delta t = b + a \cdot x$. Set $\vec{P} = \vec{o} + T(x) \vec{D}$. [S1]
            *   Increment:
                $$ \vec{P} \leftarrow \vec{P} + \Delta t \cdot \vec{D} $$
                $$ \Delta t \leftarrow \Delta t + a $$
                $$ x \leftarrow x + 1 $$
            [S1]

**Edge Cases and Pitfalls:**
*   **Duplicate Samples at Macrocell Boundaries:** When entering a new macrocell, the sampling sequence might restart or overlap. To avoid duplicate samples, store the greatest $t$ already sampled. If the computed $T(x)$ for the new macrocell entry is less than the stored max $t$, skip or adjust $x$ [S1].
*   **Precision:** The quadratic growth of step size can lead to large steps far from the camera. Ensure the volume exit condition is robust [S1].
*   **Nyquist Limit:** Using $sk < 0.5$ satisfies the Nyquist limit theoretically but is computationally expensive. $sk=4$ is a practical default [S1].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of samples per ray. The number of samples is generally higher than uniform sampling for near-field features but lower for far-field features, balancing accuracy and performance.
*   **Memory Complexity:** $O(1)$ per ray for state variables.
*   **Strengths:** Respects the Nyquist frequency of the image plane, reducing aliasing artifacts for structures close to the viewpoint. Simple incremental update makes it efficient on GPUs [S1].
*   **Limitations:** Requires computation of ray differentials ($\partial \vec{D} / \partial x$) per pixel. Adaptive sampling based on local gradients is not used directly due to poor thread coherence on GPUs, though macrocell-based adaptation is possible [S1].
*   **When to Use:** High-quality volume rendering where aliasing is a concern, particularly for large volumes viewed from varying distances.
*   **When Not to Use:** Real-time applications with strict performance constraints where uniform sampling is sufficient, or when the volume has no high-frequency features near the camera.

## Background
Differential sampling was introduced in the context of volume ray casting with peak finding, building on the concept of ray differentials originally formulated by Igehy [S1]. The method addresses the issue that regular sampling in world-space undersamples features close to the viewpoint relative to those further away, due to the convolution of the volume rendering integral during the transformation from world-space to image-space [S1]. It was implemented in a GLSL fragment shader framework for OpenGL [S1].

## Sources
[S1] Volume Ray Casting with Peak Finding and Differential ..., https://www.sci.utah.edu/~knolla/avrc.pdf, fetched 2026-10-27
