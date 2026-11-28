# Light Director Frame Update

Also called: Light Director, Light Management

## Overview
The Light Director Frame Update is the per-frame management routine for the Visible Light Collection system. It maintains a fixed-size pool of light slots, promoting static chunk emitters based on camera proximity and managing dynamic light sources. It constructs the final list of `PointLight` structures passed to the renderer, handling epoch tracking for shadow consistency and flicker effects. It differs from sibling methods by focusing on slot lifecycle management (promotion, eviction, and dynamic tracking) rather than the geometric culling or radiometric calculation itself.

## How it works

### Inputs and Outputs
- **Inputs**:
  - `dt`: Time delta since the last frame.
  - `camera_pos`: Current camera position in world space.
  - `static_emitters`: List of all chunk emitters in the world.
  - `dynamic_lights`: List of currently active dynamic light sources.
  - `slots`: The persistent array of light slot structures from the previous frame.
  - `MAX_LIGHTS`: Total capacity of the `slots` array.
  - `MAX_DYNAMIC`: Maximum number of dynamic lights allowed in the final list.
  - `PROMOTE_RANGE`: Distance threshold for static emitter promotion.
- **Outputs**:
  - `final_lights`: A list of `PointLight` structures ready for rendering.
  - `slots`: Updated slot state for the next frame.

### Data Structures

**Slot**
A fixed-size structure representing a managed light slot.
- `key`: `Key::Block(pos)` identifying the chunk/block.
- `epoch`: Integer counter for shadow consistency.
- `last_pos`: `Vec3` position of the light in the previous frame.
- `emitter`: Reference or data to the static emitter source.
- `is_active`: Boolean flag indicating if the slot is currently holding a valid light.

**PointLight**
The output structure for the renderer.
- `position`: `Vec3` world position.
- `color`: `Vec3` RGB intensity.
- `intensity`: Float scalar.
- `epoch`: Integer for shadow map consistency.
- `shadows`: Boolean flag.
- `suppress`: `Vec2` (x, y) values for light suppression/culling.

### Parameters
- `PROMOTE_RANGE`: Maximum distance from camera for a static emitter to be considered for promotion. Value not explicitly defined in the sources [S1, unofficial].
- `MAX_LIGHTS`: Total number of light slots available. Value not explicitly defined in the sources [S1, unofficial].
- `MAX_DYNAMIC`: Maximum number of dynamic lights allowed. Value not explicitly defined in the sources [S1, unofficial].

### Algorithm Steps

1. **Advance Clock**: Increment the global simulation clock by `dt` [S1, unofficial].
2. **Collect Static Candidates**:
   - Iterate through all chunk emitters.
   - Calculate distance `d` from emitter position to `camera_pos`.
   - If `d < PROMOTE_RANGE`, add `(Key::Block(pos), emit / (1.0 + d))` to a `candidates` list [S1, unofficial].
3. **Calculate Static Slot Count**:
   - Determine the number of static slots to fill: `n_static = MAX_LIGHTS - min(dynamic_lights.len(), MAX_DYNAMIC)` [S1, unofficial].
4. **Promote Static Lights**:
   - Call `promote(slots, candidates, n_static)` to update the `slots` array. This function selects the best candidates from `candidates` to fill or replace entries in `slots` based on priority (likely distance or intensity) [S1, unofficial].
5. **Collect Active Keys**:
   - Gather all keys from `slots` that are marked active.
   - Gather all keys from `dynamic_lights`.
   - Combine these into a set of `active_keys` [S1, unofficial].
6. **Prune Slots**:
   - Iterate through `slots`. For any slot whose `key` is not in `active_keys`, mark it as inactive or clear it. Retain `epoch` and `last_pos` only for active keys [S1, unofficial].
7. **Construct Static PointLights**:
   - Iterate through `slots`. For each active block key:
     - Retrieve the `emitter`.
     - Calculate flicker: If `emitter.color[0] > emitter.color[2]`, generate a flicker value using a phase derived from the block coordinates [S1, unofficial].
     - Construct a `PointLight` using `static_light()` parameters.
     - Set `key`, `epoch`, and `shadows` flags [S1, unofficial].
8. **Update Dynamic Lights**:
   - Iterate through `dynamic_lights`.
   - Check if the light has moved: if `distance(current_pos, last_pos) > 0.15`:
     - Increment `epoch`.
     - Update `last_pos` to `current_pos` [S1, unofficial].
9. **Construct Dynamic PointLights**:
   - For each dynamic light:
     - Use `last_pos` (or current pos if first frame) as position.
     - Apply flicker if `color[0] > color[2]` [S1, unofficial].
     - Set `suppress` to `(0.0, 1.0)` [S1, unofficial].
10. **Finalize List**:
    - Combine the constructed static and dynamic `PointLight` lists into `final_lights` [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def light_director_frame_update(dt, camera_pos, static_emitters, dynamic_lights, slots, MAX_LIGHTS, MAX_DYNAMIC, PROMOTE_RANGE):
    # 1. Advance clock
    global_clock += dt

    # 2. Collect static candidates
    candidates = []
    for emitter in static_emitters:
        d = distance(emitter.pos, camera_pos)
        if d < PROMOTE_RANGE:
            # emit is the emitter's base intensity
            candidates.append((emitter.key, emitter.emit / (1.0 + d)))

    # 3. Calculate n_static
    dynamic_count = min(len(dynamic_lights), MAX_DYNAMIC)
    n_static = MAX_LIGHTS - dynamic_count

    # 4. Promote static lights
    # promote updates slots in-place, selecting best candidates
    promote(slots, candidates, n_static)

    # 5. Collect active keys
    active_keys = set()
    for slot in slots:
        if slot.is_active:
            active_keys.add(slot.key)
    for light in dynamic_lights:
        active_keys.add(light.key)

    # 6. Prune slots (retain epoch/last_pos for active keys)
    for slot in slots:
        if not slot.is_active or slot.key not in active_keys:
            slot.is_active = False
        # else: retain epoch and last_pos

    # 7. Construct Static PointLights
    final_lights = []
    for slot in slots:
        if not slot.is_active:
            continue
        
        emitter = get_emitter(slot.key)
        flicker = 1.0
        if emitter.color[0] > emitter.color[2]:
            phase = get_phase_from_block_coords(slot.key.pos)
            flicker = calculate_flicker(phase)
        
        light = PointLight()
        light.position = slot.key.pos
        light.color = emitter.color * flicker
        light.intensity = emitter.emit / (1.0 + distance(slot.key.pos, camera_pos))
        light.epoch = slot.epoch
        light.shadows = True
        light.suppress = (0.0, 1.0) # Static lights may have specific suppression logic not detailed here
        final_lights.append(light)

    # 8. Update Dynamic Lights
    for light in dynamic_lights:
        if distance(light.pos, light.last_pos) > 0.15:
            light.epoch += 1
            light.last_pos = light.pos

    # 9. Construct Dynamic PointLights
    for light in dynamic_lights:
        pl = PointLight()
        pl.position = light.last_pos
        pl.color = light.color
        if light.color[0] > light.color[2]:
            # Dynamic flicker logic
            pl.color *= calculate_flicker(light.epoch)
        pl.epoch = light.epoch
        pl.shadows = True
        # Pitfall: Dynamic lights are not included in flood-fill calculation
        pl.suppress = (0.0, 1.0)
        final_lights.append(pl)

    return final_lights, slots

def promote(slots, candidates, n_static):
    # Implementation details of promote are not provided in the source snippet [S1, unofficial].
    # It is assumed to select the top n_static candidates and assign them to slots.
    # [G] Standard greedy selection: sort candidates by intensity/distance, 
    # assign to empty or lowest-priority slots.
    pass
```

### Edge Cases and Pitfalls
- **Dynamic Light Suppression**: Dynamic lights are not included in the flood-fill calculation used for static lights. Therefore, their `suppress` value is hardcoded to `(0.0, 1.0)` [S1, unofficial]. This may lead to incorrect light culling or shadow artifacts if the flood-fill logic is relied upon for dynamic lights [S1, unofficial].
- **Precision**: Distance checks for dynamic light movement use a threshold of `0.15` [S1, unofficial]. Floating-point precision issues may cause flickering epochs if positions are very close to this threshold.
- **Boundary Conditions**: If `dynamic_lights.len() > MAX_DYNAMIC`, only the first `MAX_DYNAMIC` are considered, potentially ignoring valid lights. If `n_static < 0`, the logic fails (not handled in sources) [S1, unofficial].

## Operating characteristics

### Time and Memory Complexity
- **Time**: $O(N + M)$, where $N$ is the number of static emitters and $M$ is the number of dynamic lights. The iteration over static emitters and dynamic lights is linear. The `promote` function complexity depends on implementation but is typically $O(N \log N)$ or $O(N)$ with a heap [S1, unofficial].
- **Memory**: $O(K)$, where $K$ is `MAX_LIGHTS`. The slots array is fixed size. The `candidates` list is $O(N)$ [S1, unofficial].

### Measured Performance
- Not given in the sources.

### Scaling
- Performance scales with the number of visible chunk emitters and dynamic lights. `MAX_LIGHTS` caps the output size, ensuring consistent renderer load [S1, unofficial].

### Strengths and Limitations
- **Strengths**: Fixed slot count ensures predictable memory usage. Epoch tracking allows for efficient shadow map reuse.
- **Limitations**: Dynamic lights are excluded from flood-fill suppression, potentially causing visual artifacts. `PROMOTE_RANGE` and `MAX_LIGHTS` are hardcoded or not explicitly defined, limiting configurability [S1, unofficial].

### When to Use
- Use when a fixed number of lights is required for performance stability.
- Use when static chunk-based lighting is dominant.

### When Not to Use
- Do not use if dynamic light suppression via flood-fill is critical.
- Do not use if the number of lights needs to scale dynamically without a hard cap.

## Background

### Origin
The Light Director Frame Update is part of the lighting system in Wildforge, a Minecraft-alpha-style voxel game written in Rust [S1, unofficial].

### History
The method is described in the context of the Wildforge engine's implementation, which features hot-reloadable texture packs and procedural sound [S1, unofficial].

### Usage
Used in the Wildforge game engine for managing per-frame light updates [S1, unofficial].

## Sources

[S1] A Minecraft-alpha-style voxel game written in Rust with a custom engine, https://github.com/dollspace-gay/Wildforge, fetched 2026-10-26 (unofficial)
