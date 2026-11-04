# Differentiable DDA Loop

Also called: Differentiable DDA

## Overview
A differentiable Digital Differential Analyzer (DDA) loop that traverses a grid along a ray, invoking a callback function for each visited cell [S1, unofficial]. It sets itself apart from standard DDA implementations by maintaining differentiability through the traversal state, allowing gradients to flow back through the ray marching steps [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Ray direction `ray_d2` (normalized or scaled), grid resolution `grid_res`, initial ray origin (implicit in `p0`), and a callback function `func`.
*   **Outputs:** The state returned by the final callback invocation.

**Data Structures:**
*   `active`: Boolean array indicating which rays are still valid.
*   `state`: User-defined state object passed to the callback.
*   `dt_v`: Array of step sizes to the next grid line for each active ray.
*   `p0`: Current position of the ray.
*   `pi`: Current grid index (integer coordinates).
*   `t_rem`: Remaining ray length to traverse.
*   `abs_rcp_d2`: Precomputed absolute reciprocal of the ray direction components.

**Numbered Steps:**

1.  **Loop Initialization:** Initialize `active` to all true. Initialize `dt_v` based on ray direction and grid spacing. Initialize `p0` to the ray origin. Initialize `pi` to the starting grid cell. Initialize `t_rem` to the total ray length. Initialize `state` to the user's initial state.

2.  **Step Selection:** Compute the step size for this iteration:
    $$ dt = \min(\min(dt_v), t_{rem}) $$
    Identify which grid lines are hit at this step:
    $$ mask = (dt_v == dt) $$

3.  **Position Update:** Compute the new position using the Fused Multiply-Add (FMA) operation:
    $$ p_1 = \text{fma}(ray\_d2, dt, p_0) $$

4.  **Callback Invocation:** Invoke the user callback function:
    $$ state, cont = func(state, \text{ArrayNu}(pi), p_0, p_1, active \ \& \ (dt > 0)) $$
    Note: The callback is only invoked if `dt > 0` to handle the edge case where the ray starts exactly on a grid line [S1, unofficial].

5.  **Grid Index Update:**
    *   Update `dt_v` for active rays:
        $$ dt_v[active] = \text{select}(mask, abs\_rcp\_d2, dt_v - dt) $$
    *   Update `p0` for active rays:
        $$ p_0[active] = \text{fma}(ray\_d2, dt, p_0) $$
    *   Reset `p0` components for rays that crossed a grid line boundary:
        $$ p_0[mask \ \& \ active] = \text{select}(ray\_d2 \ge 0, \text{Float}(0), \text{Float}(1)) $$
    *   Increment `pi` for rays that crossed a grid line boundary:
        $$ pi[mask \ \& \ active] += \text{step} $$
        Where `step` is +1 or -1 depending on the direction of `ray_d2`.
    *   Update `t_rem` for active rays:
        $$ t_rem[active] = t_rem - dt $$

6.  **Activity Update:** Update the `active` mask for the next iteration:
    $$ active[active] \ \&= \ \text{all}((pi \ge 0) \ \& \ (pi < \text{ArrayNi}(grid\_res))) \ \& \ (t_{rem} > 0) \ \& \ cont $$
    A ray becomes inactive if it goes out of bounds, runs out of length, or if the callback returns `cont=False`.

**Pitfalls:**
*   **Starting on a grid line:** If the ray starts exactly on a grid line, `dt` can be 0. The code handles this by checking `dt > 0` before invoking the callback [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def differentiable_dda_loop(ray_d2, grid_res, func, initial_state, ray_origin, ray_length):
    # Initialization
    active = True
    state = initial_state
    abs_rcp_d2 = abs(1.0 / ray_d2)
    dt_v = compute_initial_dt_v(ray_origin, ray_d2, abs_rcp_d2) # Implementation detail [G]
    p0 = ray_origin
    pi = compute_initial_grid_index(ray_origin, ray_d2) # Implementation detail [G]
    t_rem = ray_length
    
    while active:
        # Step Selection
        dt = min(min(dt_v), t_rem)
        mask = (dt_v == dt)
        
        # Position Update
        p1 = fma(ray_d2, dt, p0)
        
        # Callback Invocation
        # Only invoke if dt > 0 to handle starting on grid line
        if dt > 0:
            state, cont = func(state, pi, p0, p1, active)
        else:
            cont = True # Assume continue if no step taken
            
        # Grid Index Update
        dt_v = select(mask, abs_rcp_d2, dt_v - dt)
        p0 = fma(ray_d2, dt, p0)
        p0 = select(mask & active, ray_d2 >= 0, 0.0, 1.0)
        pi = pi + step * (mask & active) # step is +1 or -1 based on ray_d2 sign
        t_rem = t_rem - dt
        
        # Activity Update
        out_of_bounds = (pi < 0) | (pi >= grid_res)
        active = active & ~out_of_bounds & (t_rem > 0) & cont
        
    return state
```

## Subtopics

## Operating characteristics
*   **Time Complexity:** Linear in the number of grid cells visited by the ray.
*   **Memory Complexity:** Constant per ray, proportional to the grid resolution for storing `dt_v`, `p0`, `pi`, `t_rem`.
*   **Strengths:** Differentiable, allowing gradients to be computed with respect to ray parameters or grid properties.
*   **Limitations:** Requires careful handling of edge cases like starting on grid lines. Performance may be impacted by the overhead of differentiable operations.
*   **When to use:** When gradients through the ray traversal are needed, e.g., in differentiable rendering or optimization.
*   **When not to use:** When performance is critical and differentiability is not required; standard DDA is likely faster.

## Background
The Differentiable DDA Loop is presented in the context of differentiable ray tracing, specifically within the `differentiable_xrt` repository [S1, unofficial]. It builds on the principles of the Digital Differential Analyzer (DDA) traversal [2323-dda.md] but extends it to support gradient computation.

## Sources
[S1] HaouchatY/differentiable_xrt, https://github.com/HaouchatY/differentiable_xrt, fetched 2026-10-27 (unofficial)
