# Budget Plans

Also called: chunk budget policies

## Overview
Budget Plans provide a system for managing chunk loading policies in chunked volume rendering, replacing the single octree-focused policy with presets for exploration, whole-volume figures, and frame-rate holding [S1, unofficial]. They solve the problem of rigid streaming configurations by allowing mid-session switching and re-folding of load options, ensuring that explicit application preferences and current state constraints (like minimum detail levels) are respected [S1, unofficial]. Unlike sibling methods that focus on dynamic generation radii or scanner-based unloading, Budget Plans shape the structural budget of the octree itself, determining the density and level-of-detail distribution of loaded chunks [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A `BudgetPlan` object or preset name (`focus`, `uniform`, `interactive`), current scene state (minLevel, halo), and optional explicit plan overrides.
- **Output:** A resolved `BudgetPlan` configuration that dictates chunk loading behavior, including grid dimensions for lattice plans or octree density parameters.

**Data Structures:**
- `BudgetPlan`: Contains fields for `radius` (scalar or 'volume'), `minLevel`, `halo`, and potentially `gridDims` for lattice variants.
- **Preset Precedence (Lowest to Highest):**
    1. Focus baseline
    2. Named preset (`focus`, `uniform`, `interactive`)
    3. Explicit plan object
    4. Individual options

**Steps:**
1. **Resolve Preset:** Determine the base budget plan based on the input. If a named preset is provided, it overrides the focus baseline. If an explicit plan object is provided, it overrides the preset. Individual options override the plan object.
2. **Apply Radius:** If `radius` is set to `'volume'`, enable a uniform detail shrink pass. This pass only affects levels beyond the specified radius and converges on the finest uniform level that fits within the budget [S1, unofficial].
3. **Integrate State:** Retrieve `minLevel` and `halo` from the current scene state. These values are applied to the plan to prevent the budget switch from undoing a previously set `setMaxDetail` or undercutting a wider kernel already streamed [S1, unofficial].
4. **Handle Lattice (if applicable):** If the plan uses an exact lattice grid (see Subtopics), set `gridDims` and `stride` on the plan. Ensure `chunkAtVoxel` can index this lattice. If a brick would exceed `deviceLimit`, grow the axis and clamp the oversize texture via `emitBrick` [S1, unofficial].
5. **Switch Mid-Session:** Call `setBudgetPlan` to apply the new plan. This triggers a re-fold over load options, ensuring that app-pinned knobs still win over automatic adjustments [S1, unofficial].

**Parameters:**
- **Presets:** `focus`, `uniform`, `interactive` [S1, unofficial].
- **Radius:** `'volume'` enables uniform detail shrink; otherwise, a scalar value [S1, unofficial].
- **Target Frame Time:** `targetFrameMs` is resolved and readable but currently has no active effect on the budgeting logic [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Chunk 0 Eviction:** In WebGL2, evicting chunk 0 can blank the entire volume because `draw()` early-returns on null `volumeTexture`/`volumeGradientTexture`, which are best-effort aliases of chunk 0 [S1, unofficial]. WebGPU handles this by falling back to `vol.pickSampler` (mm-space, backed by the coarse floor) [S1, unofficial].
- **Unreachable Counts:** Using scalar `cellEdge` sizing for bricks can lead to unreachable block counts (e.g., 4 or 9 on `hoa_heart` L5) due to the octree pyramid structure [S1, unofficial]. Use `gridDims` for exact lattice control [S1, unofficial].

## Subtopics

### [Block Loading Badges](6011-block-loading-badges.md)
Block Loading Badges provide per-pane visibility into the status of GPU upload queues for chunk-based volume rendering. They solve the problem of opaque streaming progress by tracking the resident count movement of fetched blocks, distinguishing actual data acquisition from planned requests. Unlike general budget plans that shape octree structure, badges specifically monitor the GPU residency cap and upload queue depth, ensuring the UI reflects only the blocks the stream actually requested and fetched.

### [Capacity Handling](14405-capacity-handling.md)
Capacity Handling is the procedure that converts continuous block retention quotas into integer budgets $K_b$ that sum exactly to a global target $K$, while respecting individual block capacities $n_b$. It solves the problem of discretizing soft allocation scores into hard chunk limits for out-of-core streaming, ensuring the total retained Gaussian count matches the budget without exceeding per-block storage limits. It differs from sibling methods like [Block Loading Badges](6011-block-loading-badges.md) by focusing on the mathematical conversion of smoothed diagonal scores into strict integer counts rather than dynamic loading flags.

### Exact Lattice Grid
Exact Lattice Grid replaces octree-based brick sizing with an exact lattice defined by `gridDims`, allowing specific block counts that were previously unreachable due to scalar `cellEdge` sizing [S1, unofficial]. It solves the problem of rigid block count stepping in octree structures. Steps: 1. Set real `gridDims`/stride on plan so `chunkAtVoxel` indexes it. 2. Axis grows when brick would exceed `deviceLimit` (emitBrick clamps oversize texture). 3. Callers read built plan's own `gridDims` back [S1, unofficial]. Pitfall: Previous `chunkVolumeMultiLOD` sized bricks from scalar `cellEdge`, so counts stepped 2, 8, 12, 16, 27, 4 on `hoa_heart` L5; 4 or 9 were unreachable. `maxBricks` only coarsened by raising level floor, fighting level control [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** The `uniform` preset with `radius: 'volume'` converges on the finest uniform level that fits, which is suitable for whole-volume figures [S1, unofficial]. The `focus` preset is optimized for exploration [S1, unofficial].
- **Strengths:** Allows mid-session switching without losing user-set constraints (`minLevel`, `halo`). Provides presets for different use cases (exploration, static view, interactive rotation).
- **Limitations:** `targetFrameMs` is not yet functional. Chunk 0 eviction can crash the volume in WebGL2 if not handled by fallback sampling [S1, unofficial].
- **When to use:** Use `focus` for exploration, `uniform` for static whole-volume views, and `interactive` for frame-rate holding during rotation [S1, unofficial]. Use Exact Lattice Grid when specific block counts are required [S1, unofficial].

## Background
Budget Plans were introduced to address the limitations of a single octree-focused streaming policy, which was suitable for exploration but inadequate for whole-volume figures or frame-rate holding [S1, unofficial]. The design doc is located at `docs/budget-plans.md`, with remaining streaming work tracked in `docs/streaming-todos.md` [S1, unofficial]. The Exact Lattice Grid variant was added to allow specific block counts unreachable by octree pyramid structures [S1, unofficial].

## Sources
[S1] feat(niivue): streamed multi-LOD rendering fixes, LOD compensation, budget plans, and block inspection, https://github.com/niivue/mono/pull/138, fetched 2026-10-27 (unofficial)
