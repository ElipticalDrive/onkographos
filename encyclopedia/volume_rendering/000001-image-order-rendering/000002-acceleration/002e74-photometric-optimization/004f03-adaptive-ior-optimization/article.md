# Adaptive IOR Optimization
Also called: IOR Search Algorithm
## Overview
Adaptive IOR Optimization is a training procedure for neural radiance fields that iteratively searches for the optimal Index of Refraction (IOR) to minimize the reconstruction error of transparent foreground objects [S1]. It solves the problem of unknown or inaccurate IOR values in refractive scene reconstruction by decoupling background training from foreground IOR selection [S1].
## How it works
### Inputs and Outputs
**Inputs:**
*   $F_\theta$: A multi-layer perceptron (MLP) mapping position $x$ and direction $d$ to density $\sigma$ and color $c$ [S1].
*   $[IOR_{min}, IOR_{max}]$: The initial search interval for the Index of Refraction [S1].
*   $k$: The number of segments to divide the IOR interval into during each search step [S1].
*   $M$: A mesh representing the transparent object used for ray tracing [S1].
*   $r = \{r_1, \dots, r_n\}$: Foreground camera rays [S1].
*   $c_{label} = \{c_1, \dots, c_n\}$: Ground truth pixel colors for the foreground [S1].

**Outputs:**
*   $IOR_{best}$: The optimal Index of Refraction found [S1].

### Data Structures
*   `IOR_list`: An array of $k$ evenly spaced values within the current search interval [S1].
*   `PSNR_max`: A scalar initialized to $-\infty$ to track the best performance metric [S1].
*   `IOR_best`: A scalar initialized to $IOR_{min}$ to store the current best IOR candidate [S1].

### Steps
1.  **Background Training**: Train the background radiance field using only pixels with a mask value of 0 [S1]. The background is assumed to be unaffected by refraction [S1].
2.  **Iterative IOR Search**: Divide the current IOR range $[IOR_{min}, IOR_{max}]$ into $k$ equal parts [S1]. For each part, calculate the emergent light rays after refraction $r_{refr}$ using the transparent object mesh $M$ [S1]. Render the color $c_{IOR}$ for each pixel using the radiance field and the refracted rays [S1]. Calculate the PSNR between the rendered color $c_{IOR}$ and the ground truth $c_{label}$ [S1]. Select the best IOR:
    $$ IOR_{best} = \arg\max_{IOR} PSNR(c_{IOR}(r), c_{label}(r)) $$
    [S1] Narrow the search interval to $[IOR_{best} - \Delta, IOR_{best} + \Delta]$ where $\Delta = (IOR_{max} - IOR_{min}) \times 0.4 / 2$ [S1]. Repeat the search with the new interval until convergence [S1].
3.  **Joint Training**: Once the optimal IOR is determined, train both foreground and background rays simultaneously using $IOR_{best}$ [S1]. This step ensures that rays appearing only in the foreground are properly integrated into the radiance field [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_ior_optimization(F_theta, IOR_min, IOR_max, k, M, rays, c_label):
    """
    Iteratively searches for the optimal IOR.
    [S1] Algorithm 1 from source, adapted to Python conventions.
    """
    IOR_best = IOR_min
    PSNR_max = -float('inf')
    
    current_IOR_min = IOR_min
    current_IOR_max = IOR_max
    
    # Loop until convergence or max iterations (convergence criterion not specified in source)
    while True:
        # Step 1: Divide range into k equal parts
        IOR_list = [current_IOR_min + i * (current_IOR_max - current_IOR_min) / k for i in range(k)]
        
        # Step 2: Search for best IOR in current interval
        for IOR_now in IOR_list:
            # Calculate emergent light rays after refraction
            # [G] Standard refraction ray tracing through mesh M with IOR_now
            r_refr = ray_tracing(rays, M, IOR_now)
            
            # Render color for each pixel
            # [G] Standard volume rendering integration using F_theta and refracted rays
            c_IOR = render(F_theta, r_refr)
            
            # Calculate PSNR
            # Note: Source pseudocode calculates sum of squared errors but names it PSNR_IOR.
            # We compute standard PSNR to match the explicit formula and step description [S1].
            mse = sum((c_label[i] - c_IOR[i])**2 for i in range(len(c_label))) / len(c_label)
            if mse == 0:
                psnr = float('inf')
            else:
                psnr = 10 * (1 / mse)  # Simplified PSNR assuming normalized pixel range [0,1]
            
            if psnr > PSNR_max:
                PSNR_max = psnr
                IOR_best = IOR_now
        
        # Step 3: Narrow search interval
        # Source formula: (IORmax - IORmin) * 0.4 / 2
        # Applied to current bounds to maintain logical interval contraction
        delta = (current_IOR_max - current_IOR_min) * 0.4 / 2
        
        new_IOR_min = IOR_best - delta
        new_IOR_max = IOR_best + delta
        
        # Boundary checks
        new_IOR_min = max(new_IOR_min, IOR_min)
        new_IOR_max = min(new_IOR_max, IOR_max)
        
        # Check for convergence (interval size is negligible)
        if new_IOR_max - new_IOR_min < 1e-6:
            break
            
        current_IOR_min = new_IOR_min
        current_IOR_max = new_IOR_max
        
    return IOR_best
```

### Edge Cases and Pitfalls
*   **Background Assumption**: The method assumes the background is unaffected by refraction [S1].
*   **Metric Consistency**: The source text mentions minimizing MSE but provides a formula maximizing PSNR, and the pseudocode calculates sum of squared errors while comparing against `PSNR_max` [S1]. Implementers should ensure consistency between the metric calculated and the optimization goal [S1].
*   **Convergence Criterion**: The exact stopping condition for the iterative search is not given in the sources [S1].
*   **Segment Count**: The source text specifies dividing the interval into ten equal parts, suggesting $k=10$ as a typical value [S1].
## Operating characteristics
*   **Time Complexity**: The search step is $O(k \cdot N)$ per iteration, where $N$ is the number of rays [S1]. The source claims a rendering time reduction of 70–80% depending on the proportion of the foreground [S1].
*   **Memory Complexity**: Not given in the sources.
*   **Strengths**: Decouples background and foreground training, simplifying the optimization landscape [S1]. Does not require gradient computation with respect to IOR, avoiding potential non-differentiability issues in refraction [S1].
*   **Limitations**: Relies on a discrete search, which may be slower than gradient-based methods if high precision is required [S1]. Performance depends on the initial IOR range and the number of segments $k$ [S1].
*   **When to use**: When the IOR of the transparent object is unknown and cannot be easily estimated [S1]. When the background is static and non-refractive [S1].
*   **When not to use**: When the background contains refractive elements [S1].
## Background
The method was proposed in "An Index of Refraction Adaptive Neural Refractive Radiance Field for Transparent Scenes" published in *Electronics* in 2025 [S1]. It builds upon the work of Pan et al. [45] and Sun et al. [46], which established that background radiance fields are largely unaffected by refraction [S1]. The algorithm is designed to adaptively optimize the IOR during the training process, addressing the difficulty of obtaining accurate IOR values in practical applications [S1].
## Sources
[S1] An Index of Refraction Adaptive Neural Refractive Radiance Field for Transparent Scenes, https://doi.org/10.3390/electronics14061073, fetched 2026-10-29
