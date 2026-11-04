# Budget Plans

Also called: chunk budget policies

## Overview
Budget Plans provide a system for managing chunk loading policies in chunk-based volume streaming, replacing the previous single octree-focused policy with configurable presets for exploration, whole-volume visualization, and frame-rate maintenance [S1, unofficial]. This method allows an application to switch loading strategies mid-session to balance visual fidelity against performance constraints, addressing the limitation where a single policy was "right for exploring and wrong for a whole-volume figure or for holding a frame rate while rotating" [S1, unofficial]. It sets itself apart from sibling methods like [Dynamic Chunk Generation with Radius](4002-dynamic-chunk-generation-with-radius.md) by focusing on high-level policy presets and precedence rules rather than geometric radius calculations, and from [Chunk Loading/Unloading Scanner](4404-chunk-loading-unloading-scanner.md) by managing the *intent* of the load queue rather than the mechanical scanning of the spatial hierarchy.

## How it works
**Inputs and Outputs:**
*   **Inputs:** Current chunk state (minLevel, halo), a named preset string ('focus', 'uniform', 'interactive'), or an explicit plan object.
*   **Outputs:** A resolved `BudgetPlan` object containing parameters for the chunk loader, including radius, minLevel, and halo settings.

**Data Structures:**
*   `BudgetPlan`: An object containing loading parameters. Specific fields depend on the variant (e.g., `gridDims` for Exact Lattice Grid).
*   Presets: Three predefined configurations:
    1.  `focus`: Baseline octree-focused policy.
    2.  `uniform`: Uniform detail shrink pass.
    3.  `interactive`: Frame-rate holding policy.

**Parameters:**
*   **Presets:** `focus`, `uniform`, `interactive` [S1, unofficial].
*   **Precedence (lowest to highest):**
    1.  Focus baseline [S1, unofficial].
    2.  Named preset [S1, unofficial].
    3.  Explicit plan object [S1, unofficial].
    4.  Individual options [S1, unofficial].
*   **`targetFrameMs`:** Resolved and readable, but currently inert (nothing acts on it yet) [S1, unofficial].

**Steps:**
1.  **Resolve Plan:** Call `resolveBudgetPlan` with the desired input. The function applies the precedence rules to determine the final configuration. If a named preset is used, it overrides the focus baseline but is overridden by explicit options [S1, unofficial].
2.  **Switch Plan:** Call `setBudgetPlan` to apply the resolved plan mid-session. This function "re-folds over load options" to update the streaming state [S1, unofficial].
3.  **Preserve State:** During the switch, `minLevel` and `halo` are taken from the current state. This prevents the switch from undoing a user-set `setMaxDetail` or undercutting a wider kernel already streamed [S1, unofficial].
4.  **Uniform Radius:** If the `radius` parameter is set to `'volume'`, it enables a uniform detail shrink pass. This pass only affects levels beyond the specified radius, and the budget pass converges on the finest uniform level that fits within that radius [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def resolve_budget_plan(input_config, current_state):
    """
    Resolves the final budget plan based on precedence.
    input_config: dict or string. Can be 'focus', 'uniform', 'interactive', 
                  or a dict with explicit options.
    current_state: dict with 'minLevel' and 'halo'.
    """
    # Precedence 1: Focus baseline (lowest)
    plan = {
        'radius': 'default',
        'minLevel': current_state['minLevel'], # Inherited from state
        'halo': current_state['halo']          # Inherited from state
    }
    
    # Precedence 2: Named preset
    if isinstance(input_config, str):
        if input_config == 'uniform':
            plan['radius'] = 'volume' # Enables uniform detail shrink pass
        elif input_config == 'interactive':
            # Specific interactive logic not detailed in sources
            pass
        elif input_config == 'focus':
            # Remains default
            pass
        return plan

    # Precedence 3 & 4: Explicit plan object or individual options
    if isinstance(input_config, dict):
        # Override specific fields if present in input_config
        if 'radius' in input_config:
            plan['radius'] = input_config['radius']
        if 'minLevel' in input_config:
            plan['minLevel'] = input_config['minLevel']
        if 'halo' in input_config:
            plan['halo'] = input_config['halo']
            
    return plan

def set_budget_plan(new_plan, current_state):
    """
    Switches the budget plan mid-session.
    """
    # Re-fold over load options
    # Ensure minLevel and halo are preserved from current state 
    # to prevent undoing setMaxDetail or undercutting wider kernels
    final_plan = resolve_budget_plan(new_plan, current_state)
    
    # Apply final_plan to the chunk loader
    # Note: targetFrameMs is resolved but currently inert
    apply_loading_policy(final_plan)
```

**Edge Cases and Pitfalls:**
*   **State Undoing:** Switching plans must not reset `minLevel` or `halo` to defaults, as this would undo user controls like `setMaxDetail` or disrupt active streaming kernels [S1, unofficial].
*   **Inert Parameter:** `targetFrameMs` is part of the plan but does not currently influence loading behavior [S1, unofficial].

## Subtopics

### Exact Lattice Grid
Replaces octree-based brick sizing with an exact lattice defined by `gridDims`, allowing specific block counts that were previously unreachable due to scalar `cellEdge` sizing [S1, unofficial]. It sets itself apart by allowing precise control over block counts (e.g., 4, 9, 16) rather than relying on pyramid-based scalar sizing [S1, unofficial]. Choose this when specific block dimensions are required for hardware or rendering constraints.
*   **Definition:** Uses `gridDims[a]` bricks per axis, all drawn from `minLevel` [S1, unofficial].
*   **Steps:**
    1.  Set real `gridDims` and `stride` on the plan so `chunkAtVoxel` indexes it correctly [S1, unofficial].
    2.  An axis grows if a brick would exceed `deviceLimit`, because `emitBrick` clamps oversize textures, which squashes sampling [S1, unofficial].
    3.  Callers read the built plan's own `gridDims` back to verify the actual dimensions [S1, unofficial].
*   **Pitfall:** Previous `chunkVolumeMultiLOD` sized bricks from a scalar `cellEdge`, causing reachable counts to step irregularly (e.g., 2, 8, 12, 16, 27, 4 on `hoa_heart` L5), making counts like 4 or 9 unreachable [S1, unofficial]. `maxBricks` only coarsened by raising the level floor, fighting level control [S1, unofficial].

### Block Loading Badges
Per-pane block-loading badges indicating GPU upload queue status, keyed on resident count movement rather than plan total [S1, unofficial]. It sets itself apart by tracking actual stream requests rather than theoretical plan totals. Choose this for accurate UI feedback on GPU load.
*   **Definition:** Badges key on resident count moving, counting against what the stream actually asked for, not the plan total [S1, unofficial].
*   **Steps:**
    1.  Badge keys on resident count moving [S1, unofficial].
    2.  Counts against what the stream actually asked for, not plan total [S1, unofficial].
    3.  Absent/empty bricks are never fetched, and LRU evicts them [S1, unofficial].
    4.  Holds a per-episode high-water mark so the bar only moves forward [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Allows mid-session switching of loading strategies without losing user-set detail levels; supports precise block counts via Exact Lattice Grid.
*   **Limitations:** `targetFrameMs` is currently inert; Exact Lattice Grid requires careful handling of `deviceLimit` to avoid texture squashing.
*   **When to use:** Use when the application needs to switch between exploration (focus), full-volume visualization (uniform), or performance-critical rotation (interactive). Use Exact Lattice Grid when specific block counts are required.

## Background
Budget Plans were introduced in the `niivue` project to address the limitations of a single octree-focused chunk loading policy [S1, unofficial]. The design doc is located at `docs/budget-plans.md`, with remaining streaming work tracked in `docs/streaming-todos.md` [S1, unofficial]. The implementation includes fixes for chunk eviction issues (e.g., chunk 0 eviction blanking the volume in WebGL2) and adapts to both WebGL2 and WebGPU backends [S1, unofficial]. The Exact Lattice Grid variant was added to support specific block counts in demos like the Zarr demo, which uses a "Blocks control" to choose factorizations closest to cubic [S1, unofficial].

## Sources
[S1] feat(niivue): streamed multi-LOD rendering fixes, LOD compensation, budget plans, and block inspection, https://github.com/niivue/mono/pull/138, fetched 2026-10-27 (unofficial)
