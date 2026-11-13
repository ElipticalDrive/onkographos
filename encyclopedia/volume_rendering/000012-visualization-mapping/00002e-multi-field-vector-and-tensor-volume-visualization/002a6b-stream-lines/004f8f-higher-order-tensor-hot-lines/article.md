# Higher Order Tensor (HOT) Lines

Also called: Generalized Tensor Lines, HOT-Lines, Major tensor lines, Median higher order tensor lines, Minor tensor lines, Tracking Lines in Higher Order Tensor Fields

## Overview
Higher Order Tensor (HOT) Lines is a method for visualizing vector and tensor fields of arbitrary order by computing trajectories that follow the local extrema of a function defined on the unit sphere at each spatial position [S1]. It solves the problem of tracking structures in fields where second-order tensor representations fail, such as regions with fiber crossings, kissings, or T/Y-junctions in diffusion MRI data [S1]. Unlike [Stream Lines](4796-stream-lines.md) which integrate a single vector field, HOT Lines integrates direction fields derived from the maxima, minima, or saddles of a spherical harmonic expansion of the tensor, allowing for multiple distinct trajectories per voxel [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A $C^2$ continuous tensor field $T$ of arbitrary order $r$ defined on a domain $U \subseteq \mathbb{R}^3$. The field is represented via spherical harmonic coefficients.
*   **Output:** A set of curves (lines) parameterized by time $t$, representing major, minor, or median trajectories.

**Data Structures**
The symmetric higher-order tensor field is represented as a spherical harmonic expansion. For a tensor of order $l$ (where only even $l$ are used for symmetric tensors), the function $f_\sigma(g)$ on the unit sphere $S^2$ is:
$$ f_\sigma(g) = \sum_{l} \sum_{m=-l}^{l} a_l^m Y_l^m(g) $$
where $Y_l^m$ are real spherical harmonics and $a_l^m$ are real coefficients. Due to symmetry, coefficients for odd $l$ are zero. The number of real coefficients required is $(l+1)(l+2)/2$ [S1].

**Definitions**
At each position $p$, a function $f_\sigma(p): S^2 \to \mathbb{R}$ is defined [S1]:
$$ f_\sigma(p)(\theta, \phi) $$
A position $p$ is **degenerated** if there exists a point $(\theta, \phi) \in S^2$ such that the determinant of the spherical Hessian is zero [S1]:
$$ \det |\nabla_{S^2}^2 f_\sigma(p)(\theta, \phi)| = 0 $$
At a **regular point** $q$, there is a finite number $M$ of isolated maxima $(\theta_1, \phi_1), \dots, (\theta_M, \phi_M)$ [S1]. Using the implicit function theorem, unique $C^1$ functions $g_m: U_m \to S^2$ are obtained, parameterizing these maxima over neighborhoods $U_m$ [S1]:
$$ \nabla_{S^2} f_\sigma(p)(g_m(p)) = 0 $$
$$ \nabla_{S^2}^2 f_\sigma(p)(g_m(p)) \text{ is negative definite} $$
[ S1 ]

**Line Definitions**
*   **Major HOT Lines:** Curves $x_m: I_m \to U_m$ starting at $q$ with $x_m(0) = q$, integrated via:
    $$ \frac{\partial x_m(t)}{\partial t} = g_m(x_m(t)) $$
    [ S1 ]
*   **Minor HOT Lines:** Defined by substituting maxima with minima. The source states the Hessian becomes "negative definite" in this context, though mathematically minima require a positive definite Hessian; the method treats them as the dual extrema [S1].
*   **Median HOT Lines:** Lines following the direction of saddle points of $f_\sigma$ [S1].

**Special Cases**
*   **Second Order Tensors:** For a symmetric second-order tensor $\sigma_{jk}$ with eigenvalues $\lambda_i$ ($\lambda_i \ge \lambda_{i+1}$) and eigenvectors $e_i$, if non-degenerated, $f_{\sigma_{jk}}$ has two maxima in directions $e_0$ and $-e_0$. Major HOT Lines are equivalent to second-order tensor lines. A degenerated second-order tensor for major HOT Lines has $\lambda_0 = \lambda_1$. The case $\lambda_0 > \lambda_1 = \lambda_2$ is not degenerated [S1].
*   **First Order Tensors (Vectors):** Major HOT Lines are equivalent to [Stream Lines](4796-stream-lines.md). Minor HOT Lines are streamlines on the negated tensor field [S1].
*   **Pure Order Zero (Scalar):** Tensors containing only a scalar component $a_{00}$ are degenerated because scalar fields have no orientation. Tracking is impossible [S1].

**Algorithm**
The tracking process uses a main loop to manage multiple directions and a line tracking function.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def track_hot_lines(tensor_field, start_points, initial_step_size, 
                    min_step_size, max_step_size, direction_accuracy):
    """
    Main loop for tracking HOT lines.
    tensor_field: Function returning spherical harmonic coefficients or 
                  evaluating f_sigma and its derivatives at a point.
    start_points: List of initial positions.
    """
    to_track = []
    
    # Initialize tracking for all start points
    for p in start_points:
        # Get all maxima at current position p
        maxima = get_maxima_at_point(tensor_field, p)
        for direction in maxima:
            to_track.append({
                'position': p,
                'direction': direction,
                'stepsize': initial_step_size
            })
            
    while to_track:
        # Get next triple
        item = to_track.pop(0)
        track_line(
            tensor_field, 
            item['position'], 
            item['direction'], 
            item['stepsize'],
            min_step_size, 
            max_step_size, 
            direction_accuracy,
            to_track # Pass list to append new branches if needed
        )

def track_line(tensor_field, position, direction, stepsize, 
               min_step_size, max_step_size, direction_accuracy, 
               to_track):
    """
    Integrates a single HOT line.
    """
    stopped = False
    while not stopped:
        step_accepted = False
        while not step_accepted or stopped:
            # Check if stepsize is too small
            if stepsize < min_step_size:
                # Stop because degenerated point reached
                stopped = True
                break
            
            # Calculate step
            step = direction * stepsize
            
            # Get directions at position + step
            next_position = position + step
            next_directions = get_maxima_at_point(tensor_field, next_position)
            
            # Check for single direction close to current direction
            if len(next_directions) == 1:
                next_dir = next_directions[0]
                if angle_between(direction, next_dir) < direction_accuracy:
                    # Accept step
                    step_accepted = True
                    # Increase stepsize if new direction is close enough
                    if angle_between(direction, next_dir) < (direction_accuracy / 2):
                        stepsize = min(stepsize * 2, max_step_size)
                    else:
                        stepsize = min(stepsize * 1.5, max_step_size)
                    
                    # Update for next iteration
                    position = next_position
                    direction = next_dir
                else:
                    # Direction not close enough, reduce stepsize
                    stepsize /= 2
            elif len(next_directions) > 1:
                # Multiple directions close to current direction
                # Append position to degenerated points set and stop
                # (Implementation detail: log degenerated point)
                stopped = True
                break
            else:
                # No directions found (e.g., scalar region)
                stopped = True
                break
                
        if not stopped and step_accepted:
            # Record point or continue
            pass
```

**Parameters**
*   **Stepsize Management:** Lower and upper limits are required, relating to the smoothness of the data set [S1].
*   **Direction Accuracy:** Depends on the wanted tracking accuracy and the order of tensors, because higher orders can produce many maxima that are close together [S1].

**Pitfalls**
*   **Degenerated Points:** Tracking stops if the stepsize becomes too small, indicating a degenerated point where the Hessian determinant is zero or maxima vanish [S1].
*   **Multiple Directions:** If multiple directions are close to the current direction during tracking, the point is appended to the set of degenerated points and tracking stops [S1].
*   **Scalar Fields:** Pure scalar components ($a_{00}$) make tracking impossible and are treated as degenerated [S1].

## Subtopics

### HOT- Lines Fiber Tracking with Anisotropy-Based Switching
A fiber tracking algorithm that improves speed by using simple second-order tensor tracking in areas with one dominant fiber direction, and switching to higher-order tensor analysis when anisotropy is low or direction deviation is large [S1]. It loops through the grid, computing eigenvectors and anisotropy. If anisotropy is below a threshold (typically 0.2–0.3, depending on SNR) or direction deviation is too large, it processes the next step with higher-order lines; otherwise, it accepts the step using second-order tensor lines [S1]. This leverages the similarity of HOT Lines with second-order tensor lines to optimize performance in regions with single dominant directions [S1].

## Operating characteristics
*   **Complexity:** The method relies on spherical harmonic expansions. For symmetric tensors of even order $l$, the representation requires $(l+1)(l+2)/2$ real coefficients [S1]. Analytical calculation of gradients is possible, improving speed [S1].
*   **Performance:** The method is designed for efficiency, particularly when using spherical harmonic representations. It is faster than Q-Space or Q-Ball imaging which compute the Orientation Distribution Function (ODF) directly, avoiding higher data acquisition times [S1].
*   **Strengths:** Handles complex fiber configurations (crossings, kissings, T/Y-junctions) that second-order tensors fail to resolve. Unifies streamlines and tensor lines.
*   **Limitations:** Degenerated points (scalar components, isotropic regions) cause tracking to stop. Higher-order tensors can produce many close maxima, requiring careful direction accuracy tuning [S1].
*   **Use Cases:** Diffusion MRI tractography, visualization of stress fields, and other applications requiring tracking in arbitrary-order tensor fields [S1].

## Background
HOT Lines were introduced to address the limitations of second-order tensor visualization in medical imaging, specifically in diffusion MRI where fiber crossings are common [S1]. The method generalizes tensor lines to arbitrary orders using spherical harmonics, building on the mathematical properties of symmetric tensors and spherical harmonic expansions [S1]. It was developed to provide a unified framework for tracking lines in vector fields (streamlines) and higher-order tensor fields, with specific applications in understanding the connectivity and function of the human brain [S1].

## Sources
[S1] HOT- Lines: Tracking Lines in Higher Order Tensor Fields, https://doi.org/10.1109/vis.2005.48, fetched 2026-10-27
