# Volume Sculpting

Also called: Clay or wax-like sculpting

## Overview
Volume sculpting is a free-form interactive modeling technique based on the metaphor of sculpting and painting a voxel-based solid material, such as a block of marble or wood [S2, book]. It addresses the difficulty of modeling topologically complex and highly-detailed objects in most CAD systems, and enables scientists and physicians to explore inner structures of simulated or sampled datasets by gradually removing material [S1, unofficial][S2, book]. It differs from surface-based sculpting by operating directly on volume buffers, which allows sculpting actions to be localized in memory and enables localized rendering to reproject only affected pixels [S1, unofficial][S2, book].

## How it works
**Inputs and Outputs:** The method takes a source volume buffer representing the object, a tool definition (either pre-existing or generated on-the-fly), and user interaction parameters (position, orientation, scale) as inputs [S1, unofficial][S2, book]. It outputs a modified volume buffer with material removed, and a list of affected pixels for localized rendering updates [S1, unofficial][S2, book].

**Data Structures:** The object is represented as a volume buffer (a 3D array of scalar values) [S2, book]. While flat 3D arrays were used in earlier systems, multi-resolution data structures such as octrees are preferred for large datasets [S2, book]. The tool is also represented as an independent volume buffer [S1, unofficial].

**Steps:**
1. **Tool Selection and Generation:** Carving uses a pre-existing volume-sampled tool [S1, unofficial][S2, book]. Sawing generates a volume-sampled tool on-the-fly via a user interface [S1, unofficial][S2, book].
2. **Tool Positioning:** Position the tool buffer relative to the object buffer based on user input [S1, unofficial][S2, book].
3. **Sculpting Operation:** Carving performs a Boolean subtraction between the object and tool volume buffers [S1, unofficial][S2, book]. Sawing removes a whole chunk of material at once using the generated tool [S1, unofficial][S2, book].
4. **Aliasing Prevention:** Employ 3D splatting to prevent object space aliasing and achieve interactive speed during sawing [S1, unofficial][S2, book].
5. **Rendering Update:** Employ localized rendering to reproject only pixels affected by the localized sculpting actions [S1, unofficial][S2, book].

**Formulas:** The sculpting operation applies a Boolean subtraction to the scalar fields: $V_{out}(x,y,z) = \max(0, V_{obj}(x,y,z) - V_{tool}(x,y,z))$, where positive field values represent material and the null field represents empty space [G].

**Parameters:** Interaction mode is either 'carving' or 'sawing' [S1, unofficial][S2, book]. Tool parameters include position, orientation, and scale [S1, unofficial][S2, book].

**Edge Cases and Pitfalls:** Object space aliasing is prevented by 3D splatting [S1, unofficial][S2, book]. Performance relies on localized rendering for real-time interaction [S1, unofficial][S2, book]. Data structure choice impacts memory and speed, with multi-resolution structures recommended for large datasets [S2, book].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def volume_sculpt(object_buffer, tool_buffer, mode, transform):
    # Step 1: Tool Generation
    if mode == 'sawing':
        # [G] Generate volume-sampled tool on-the-fly using user interface parameters
        tool_buffer = generate_tool_on_the_fly(mode, transform)
    # Step 2: Tool Positioning
    # Position tool buffer relative to object buffer based on user input
    positioned_tool = apply_transform(tool_buffer, transform)
    # Step 3: Sculpting Operation
    # Boolean subtraction between object and tool volume buffers
    # [G] Assuming scalar field where positive values represent material
    affected_voxels = []
    for x, y, z in all_voxel_indices(positioned_tool):
        if positioned_tool[x, y, z] > 0:
            object_buffer[x, y, z] = max(0, object_buffer[x, y, z] - positioned_tool[x, y, z])
            affected_voxels.append((x, y, z))
    # Step 4: Aliasing Prevention
    # [G] 3D splatting is employed to prevent object space aliasing and achieve interactive speed
    # Step 5: Localized Rendering
    # Reproject only pixels affected by the localized sculpting actions
    affected_pixels = voxels_to_pixels(affected_voxels)
    return object_buffer, affected_pixels
```

## Operating characteristics
* **Time Complexity:** Real-time interaction is achieved through localized rendering and 3D splatting [S1, unofficial][S2, book].
* **Memory Complexity:** Depends on the data structure used; multi-resolution structures like octrees are preferred over flat 3D arrays for large datasets [S2, book].
* **Strengths:** Enables exploration of inner structures by removing material; handles topologically complex objects better than CAD [S1, unofficial][S2, book].
* **Limitations:** Requires careful handling of aliasing and performance for real-time interaction [S1, unofficial][S2, book].
* **When to Use:** When modeling topologically complex objects or exploring inner structures of volumetric datasets [S1, unofficial][S2, book].
* **When Not to Use:** Not given in the sources.

## Background
Volume sculpting was introduced for clay or wax-like sculptures [S2, book]. It is motivated by the difficulty of modeling topologically complex and highly-detailed objects in most CAD systems and the need for scientists and physicians to explore inner structures of simulated or sampled datasets by gradually removing material [S1, unofficial][S2, book]. The approach allows for real-time human interaction by localizing sculpting actions in the volume buffer and employing localized rendering [S1, unofficial][S2, book].

## Sources
[S1] PDF Volume Visualization: Principles and Advances, https://courses.cs.duke.edu/spring03/cps296.8/papers/KaufmanVolumeVisualization.pdf, fetched 2026-10-28 (unofficial)
[S2] PDF Volume Visualization and Volume Graphics, https://www3.cs.stonybrook.edu/~mueller/teaching/cse332/volvisChapter.pdf, fetched 2026-10-28 (book)
