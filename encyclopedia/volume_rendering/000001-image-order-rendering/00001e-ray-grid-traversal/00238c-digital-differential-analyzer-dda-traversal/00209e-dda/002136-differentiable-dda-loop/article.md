# Differentiable DDA Loop

Also called: Differentiable DDA

## Overview
A differentiable Digital Differential Analyzer (DDA) loop that traverses a grid along a ray, invoking a callback function for each visited cell [S1, unofficial]. It extends standard DDA traversal by maintaining differentiable state variables, allowing gradients to flow back through the ray marching steps [G].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Ray direction `ray_d2`, grid resolution `grid_res`, initial ray origin (implicit in `p0`), and a callback function `func` [S1, unofficial].
*   **Outputs:** The state returned by the final callback invocation [S1, unofficial].

**Data Structures:**
*   `active`: Boolean flag indicating whether the ray is still valid [S1, unofficial].
*   `state`: User-defined state object passed to the callback [S1, unofficial].
*   `dt_v`: Array of step sizes to the next grid line for each active ray [S1, unofficial].
*   `p0`: Current position of the ray [S1, unofficial].
*   `pi`: Current grid index (integer coordinates) [S1, unofficial].
*   `t_rem`: Remaining ray length to traverse [S1, unofficial].
*   `abs_rcp_d2`: Precomputed absolute reciprocal of the ray direction components [S1, unofficial].

**Numbered Steps:**

1.  **Loop Initialization:** Initialize `active` to true [S1, unofficial]. Initialize `dt_v` based on ray direction and grid spacing [G]. Initialize `p0` to the ray origin [S1, unofficial]. Initialize `pi` to the starting grid cell [S1, unofficial]. Initialize `t_rem` to the total ray length [S1, unofficial]. Initialize `state` to the user's initial state [S1, unofficial].
2.  **Step Selection:** Compute the step size for this iteration: `dt = dr.minimum(dr.min(dt_v), t_rem)` [S1, unofficial]. Identify which grid lines are hit at this step: `mask = dt_v == dt` [S1, unofficial].
3.  **Position Update:** Compute the new position using the Fused Multiply-Add (FMA) operation: `p1 = dr.fma(ray_d2, dt, p0)` [S1, unofficial].
4.  **Callback Invocation:** Invoke the user callback function: `state, cont = func(state, ArrayNu(pi), p0, p1, active & (dt > 0))` [S1, unofficial].
5.  **Grid Index Update:** Update `dt_v` for active rays: `dt_v[active] = dr.select(mask, abs_rcp_d2, dt_v - dt)` [S1, unofficial]. Update `p0` for active rays: `p0[active] = dr.fma(ray_d2, dt, p0)` [S1, unofficial]. Reset `p0` components for rays that crossed a grid line boundary: `p0[mask & active] = dr.select(ray_d2 >= 0, Float(0), Float(1))` [S1, unofficial]. Increment `pi` for rays that crossed a grid line boundary: `pi[mask & active] += step` [S1, unofficial]. Update `t_rem` for active rays: `t_rem[active] = t_rem - dt` [S1, unofficial].
6.  **Activity Update:** Update the `active` mask for the next iteration: `active[active] &= dr.all((pi >= 0) & (pi < ArrayNi(grid_res))) & (t_rem > 0) & cont` [S1, unofficial].

**Pitfalls:**
*   **Starting on a grid line:** If the ray starts exactly on a grid line, `dt` can be 0 [S1, unofficial]. The code handles this by selecting the minimum of `dt_v` and `t_rem`, and checking `dt > 0` before invoking the callback [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def differentiable_dda_loop(ray_d2, grid_res, func, initial_state, ray_origin, ray_length):
    # Explicit FMA and select operations to avoid library dependencies
    def fma(a, b, c): return a * b + c
    def select(cond, t, f): return t if cond else f

    # Initialization [S1, unofficial]
    active = True
    state = initial_state
    abs_rcp_d2 = [abs(1.0 / d) for d in ray_d2]
    # dt_v initialization: distance to next grid boundary [G]
    dt_v = []
    for i in range(3):
        next_grid = (pi[i] + 1) if ray_d2[i] >= 0 else pi[i]
        dt_v.append(abs((next_grid - ray_origin[i]) / ray_d2[i]))
    p0 = list(ray_origin)
    pi = [int(ray_origin[i]) for i in range(3)]
    t_rem = ray_length

    while active:
        # Step Selection [S1, unofficial]
        dt = min(min(dt_v), t_rem)
        mask = [dt_v[i] == dt for i in range(3)]

        # Position Update [S1, unofficial]
        p1 = [fma(ray_d2[i], dt, p0[i]) for i in range(3)]

        # Callback Invocation [S1, unofficial]
        if dt > 0:
            state, cont = func(state, pi, p0, p1, active and (dt > 0))
        else:
            cont = True

        # Grid Index Update [S1, unofficial]
        for i in range(3):
            if active:
                dt_v[i] = select(mask[i], abs_rcp_d2[i], dt_v[i] - dt)
                p0[i] = fma(ray_d2[i], dt, p0[i])
                if mask[i]:
                    p0[i] = select(ray_d2[i] >= 0, 0.0, 1.0)
                    step_i = 1 if ray_d2[i] >= 0 else -1
                    pi[i] += step_i
        t_rem = t_rem - dt

        # Activity Update [S1, unofficial]
        out_of_bounds = any(pi[i] < 0 or pi[i] >= grid_res[i] for i in range(3))
        active = active and not out_of_bounds and (t_rem > 0) and cont

    return state
```

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Not given in the sources.
*   **Limitations:** Not given in the sources.
*   **When to use:** Not given in the sources.
*   **When not to use:** Not given in the sources.

## Background
The Differentiable DDA Loop is presented in the context of differentiable ray tracing, specifically within the `differentiable_xrt` repository [S1, unofficial]. Not given in the sources.

## Sources
[S1] HaouchatY/differentiable_xrt, https://github.com/HaouchatY/differentiable_xrt, fetched 2026-10-27 (unofficial)
