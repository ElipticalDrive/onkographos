# VoxelDestruct Destruction

Also called: singular destruction, vox.destroy

## Overview
VoxelDestruct is a method for performing voxelated destruction on specific parts or mesh parts, returning generated debris and affected walls [S1]. It differs from its sibling [Sphere Carving](2377-sphere-carving.md) by operating on arbitrary `Part` or `MeshPart` geometries rather than spherical volumes, and from [Voxel Deformation with Spherical Falloff](4413-voxel-deformation-with-spherical-falloff.md) by focusing on discrete destruction events rather than continuous deformation [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A `focus` part (Part or MeshPart) defining the shape, size, and position of the destruction [S1]. Optional `parameters` for filtering, `voxelSize` for resolution, `debrisCount` for quantity, and `reset` timer [S1].
*   **Output:** A table containing two lists: `debris` (generated debris parts) and `walls` (affected destructible parts) [S1].

**Data Structures:**
The module maintains a `Cache` for parts to prevent instancing lag [S1]. Parts are stored in this cache if `Settings.PartCache` is true [S1]. The `hitbox` object maintains `collisions` data and runtime parts lists [S1].

**Parameters:**
*   `focus`: `Part | MeshPart`. Required. The geometry to destroy [S1].
*   `parameters`: `OverlapParams?`. Optional. Used for whitelisting/blacklisting. Example: `FilterType = Enum.RaycastFilterType.Include`, `FilterDescendantsInstances = {game.Workspace.Baseplate}` [S1].
*   `voxelSize`: `number?`. Optional.
    *   If `nil`: Intersections are not voxelized; raw intersections are returned [S1].
    *   If `0` and `Settings.Relativity` is true: `Settings.HitboxRelative` is used for distance calculation [S1].
    *   If `0` and `Settings.Relativity` is false: `Settings.VoxelDefault` is used [S1].
    *   If > `Settings.VoxelMinimum` (default 1): Used as the voxel size [S1].
*   `debrisCount`: `number?`. Optional. Quantity of debris. If `nil`, all debris is returned [S1].
*   `reset`: `number?`. Optional. Timer until destruction is cleaned up [S1].

**Settings:**
*   `Settings.VoxelMinimum = 1`: Minimum voxel size [S1].
*   `Settings.HitboxRelative = 1/3`: Relative distance to hitbox size required before destroying again [S1].
*   `Settings.GreedyMeshing = true`: Recommended to keep on [S1].
*   `Settings.RunService = false`: Recommended to keep off to avoid performance heavy frame-by-frame destruction [S1].
*   `Settings.PartCache = true`: Recommended to keep on to prevent instancing lag [S1].
*   `Settings.CachePrecreated = 10000`: Number of precreated parts in cache [S1].

**Steps:**
1.  Initialize the module: `local vox = require(game:GetService("ReplicatedStorage"):WaitForChild("VoxelDestruction"))` [S1].
2.  Call `breaker.destroy(focus, parameters, voxelSize, debrisCount, reset)` [S1].
3.  The method processes the `focus` part, optionally voxelizing it based on `voxelSize` [S1].
4.  It generates debris and identifies affected walls [S1].
5.  Return the table of debris and walls [S1].

**Pitfalls:**
*   Parts destroyed via standard `:Destroy()` cannot be returned to the cache [S1].
*   Must use the module's `.cleanup()` method to return parts to cache if `useCache` is true [S1].
*   Do not destroy debris and destructible parts when finished using them; use `.cleanup()` instead [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def destroy(focus, parameters=None, voxelSize=None, debrisCount=None, reset=None):
    """
    Performs singular destruction on a focus part.
    Returns a table of debris and a table of affected walls.
    """
    # Determine voxelization behavior
    if voxelSize is None:
        # Intersections are not voxelized and returned instead
        intersections = get_intersections(focus, parameters)
        return {"debris": intersections, "walls": intersections}
    
    # Apply voxel size logic
    if voxelSize == 0:
        if Settings.Relativity:
            voxelSize = Settings.HitboxRelative
        else:
            voxelSize = Settings.VoxelDefault
    
    # Ensure minimum voxel size
    if voxelSize < Settings.VoxelMinimum:
        voxelSize = Settings.VoxelMinimum
        
    # Voxelize the focus part
    voxels = voxelize_geometry(focus, voxelSize, parameters)
    
    # Generate debris based on debrisCount
    if debrisCount is None:
        debris = generate_all_debris(voxels)
    else:
        debris = generate_debris(voxels, debrisCount)
        
    # Identify affected walls
    walls = get_affected_walls(voxels, parameters)
    
    # Handle caching
    if Settings.PartCache:
        cache_parts(debris)
        cache_parts(walls)
        
    return {"debris": debris, "walls": walls}

def cleanup(parts):
    """
    Cleans up parts by returning them to cache or destroying them.
    """
    if isinstance(parts, Part):
        parts = [parts]
        
    for part in parts:
        if Settings.PartCache and part in cache:
            return_to_cache(part)
        else:
            part:Destroy()
```

## Subtopics

### VoxelDestruct Hitbox
A system to create hitboxes from parts, meshparts, or models that trigger destruction upon interaction [S1]. It inherits parameters from `.destroy()` [S1].
*   **Start:** `hitbox:Start()` starts the hitbox. Can be restarted while active without calling `:Stop()` first. Fires `hitbox.Started` event [S1].
*   **Stop:** `hitbox:Stop()` stops the hitbox, disconnects connections, resets `hitbox.collisions` to 0 if `revert` is true, and fires `hitbox.Stopped` event with debris and walls [S1].
*   **Fire:** `hitbox:Fire()` activates the hitbox once. If destructible parts are detected, fires `hitbox.Collision` event with debris and affected parts [S1].
*   **State:** `hitbox:IsActive()` returns boolean [S1].
*   **Runtime Data:** `hitbox:GetRuntimeParts()` and `hitbox:GetLifetimeParts()` return all created debris and affected parts from runtime [S1].
*   **Parameters:** `focus`, `parameters`, `voxelSize`, `debrisCount`, `reset` [S1]. If `voxelSize` is passed and `Relativity` is true, `Settings.HitboxRelative` is used for distance before destroying again [S1].
*   **Cleanup:** The hitbox object itself can be garbage collected, but parts must be cleaned up using `.cleanup()` [S1].

## Operating characteristics
*   **Performance:** Using `Settings.RunService = true` results in destruction every frame and is very performance heavy; it is recommended to keep this setting off [S1].
*   **Caching:** `Settings.PartCache = true` is recommended to prevent instancing lag [S1]. Parts must be cleaned up using `.cleanup()` to return them to cache; standard `:Destroy()` prevents this [S1].
*   **Greedy Meshing:** `Settings.GreedyMeshing = true` is recommended to keep on [S1].

## Background
VoxelDestruct is a module for Roblox game development, providing voxelated destruction physics with greedy meshing, hitboxes, and more [S1]. It was released as version 2.1 on the Roblox DevForum [S1].

## Sources
[S1] VoxelDestruct 2.1 | Voxelated destruction physics with ..., https://devforum.roblox.com/t/voxeldestruct-21-voxelated-destruction-physics-with-greedy-meshing-hitboxes-and-more/3044264, fetched 2026-10-30
