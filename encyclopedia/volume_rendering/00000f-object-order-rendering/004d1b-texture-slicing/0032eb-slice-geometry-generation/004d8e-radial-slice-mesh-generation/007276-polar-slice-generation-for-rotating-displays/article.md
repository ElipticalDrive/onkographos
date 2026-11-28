# Polar Slice Generation for Rotating Displays

Also called: 24-Slice Orthographic Rendering, Blender Polar Slice Generation for Rotating Volumetric Displays, Polar Coordinate Voxel Export

## Overview
Generates discrete volumetric data in 3D polar coordinates $(r, \theta, z)$ for a rotating volumetric display by rendering orthographic camera views of a boolean-intersected slice through a 3D object at 24 evenly spaced rotation angles [S1, unofficial]. It differs from multi-view stereoscopy by using a single camera and a dynamic boolean modifier to extract slices, avoiding the complexity of managing multiple cameras and their associated modifiers [S1, unofficial].

## How it works
**Inputs:**
- 3D geometry (e.g., wireframe cube) [S1, unofficial].
- Target resolution: 8x10 pixels [S1, unofficial].
- Number of slices: 24 [S1, unofficial].
- Display bit-depth: 1 (binary on/off) [S1, unofficial].

**Outputs:**
- 24 binary PNG images representing slices [S1, unofficial].
- A header file containing the parsed voxel data for embedded code [S1, unofficial].

**Data Structures:**
- Each slice is an 8x10 grid of bits.
- The final volume is a sequence of these grids indexed by $\theta$.

**Steps:**
1. Create target 3D geometry (e.g., a wireframe cube) [S1, unofficial].
2. Rotate the geometry to the desired orientation:
   - Rotate X by $45^\circ$ ($\pi/4$ radians) [S1, unofficial].
   - Rotate Y by $\phi = \arctan(1/\sqrt{2})$ radians [S1, unofficial].
3. Create a thin slice object (a cube reshaped to be slice-like) [S1, unofficial].
4. Apply a boolean modifier between the geometry and the slice object to intersect them [S1, unofficial].
5. Parent both the camera and the slice object to an empty object [S1, unofficial].
6. Configure the camera:
   - Set projection to orthographic [S1, unofficial].
   - Set resolution to 8x10 pixels [S1, unofficial].
   - Set background to black [S1, unofficial].
7. Configure the geometry material to be emissive [S1, unofficial].
8. In the compositor, add a color ramp to threshold the output to 1-bit (binary) [S1, unofficial].
9. Animate the empty's Z rotation to cycle through 24 angles. The rotation angle for frame $f$ is calculated as:
   $$ \theta_f = \frac{f}{24} \times 2\pi $$
   where $f$ is the current frame number [S1, unofficial].
10. Render the animation to produce 24 PNG slices [S1, unofficial].
11. Parse the PNGs into a header file for embedded code [S1, unofficial].

**Parameters:**
- Camera resolution: 8x10 pixels [S1, unofficial].
- Number of slices: 24 [S1, unofficial].
- Slice thickness: Thin cube (exact thickness not specified, visually adjusted) [S1, unofficial].
- Bit-depth: 1 [S1, unofficial].

**Pitfalls:**
- Continuous rotation in data generation may cause misalignment if playback speed varies with motor RPM; discrete frames are preferred [S1, unofficial].
- Slow simulation time scales can cause instabilities in fluid solvers if fluid data is used [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import bpy

# Step 1: Create geometry
cube = bpy.data.objects.new("Cube", bpy.data.meshes.new("Cube"))
bpy.context.collection.objects.link(cube)
# Apply wireframe modifier if needed
wireframe = cube.modifiers.new(name="Wireframe", type='WIREFRAME')
wireframe.thickness = 0.01

# Step 2: Rotate geometry
cube.rotation_euler = (math.pi / 4, math.atan(1 / math.sqrt(2)), 0)

# Step 3: Create slice object
slice_obj = bpy.data.objects.new("Slice", bpy.data.meshes.new("Slice"))
bpy.context.collection.objects.link(slice_obj)
# Reshape to be slice-like (e.g., scale Z down)
slice_obj.scale = (1.0, 1.0, 0.1)

# Step 4: Boolean modifier
bool_mod = cube.modifiers.new(name="Boolean", type='BOOLEAN')
bool_mod.object = slice_obj
bool_mod.operation = 'INTERSECT'

# Step 5: Parent to empty
empty = bpy.data.objects.new("Empty", None)
bpy.context.collection.objects.link(empty)
cube.parent = empty
slice_obj.parent = empty

# Step 6: Camera setup
camera = bpy.data.cameras.new("Camera")
camera_obj = bpy.data.objects.new("Camera", camera)
bpy.context.collection.objects.link(camera_obj)
camera_obj.parent = empty
camera.type = 'ORTHO'
camera.ortho_scale = 1.0 # Adjust as needed for 8x10 resolution
bpy.context.scene.camera = camera_obj

# Step 7: Material
mat = bpy.data.materials.new("Emissive")
mat.use_nodes = True
bsdf = mat.node_tree.nodes["Principled BSDF"]
bsdf.inputs['Emission Strength'].default_value = 1.0
bsdf.inputs['Color'].default_value = (1.0, 1.0, 1.0, 1.0)
cube.active_material = mat

# Step 8: Compositor threshold
bpy.context.scene.use_nodes = True
tree = bpy.context.scene.node_tree
# Add Color Ramp node and threshold logic here
# This is implementation-specific to Blender's compositor nodes

# Step 9: Animate rotation
# Set frame range for 24 frames
bpy.context.scene.frame_start = 0
bpy.context.scene.frame_end = 23

# Driver for Z rotation of empty
# Formula: (frame/24)*2*pi
# This is typically done via Blender's driver system, not pure Python loop
# For simulation purposes:
for f in range(24):
    empty.rotation_euler = (0, 0, (f / 24) * 2 * math.pi)
    bpy.context.scene.frame_set(f)
    # Render frame
    bpy.ops.render.render(write_st=True)

# Step 11: Parse PNGs
# This step is performed by an external script
```

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of slices (24) and the resolution (8x10). Rendering time depends on the complexity of the boolean intersection and the rendering engine.
- **Memory Complexity:** Low, as only 24 small images are generated and stored.
- **Strengths:** Simple setup using standard boolean operations; discrete frames allow for precise control over playback speed relative to motor RPM [S1, unofficial].
- **Limitations:** Limited to binary (on/off) voxels; resolution is fixed at 8x10 pixels; requires post-processing to convert images to embedded code format [S1, unofficial].
- **When to use:** For generating volumetric data for custom rotating volumetric LED displays with low resolution and binary output [S1, unofficial].
- **When not to use:** When continuous rotation is required without discrete frame alignment; when higher resolution or multi-bit depth is needed [S1, unofficial].

## Background
Originated from a project to create a custom RP2040-based rotating volumetric LED display prototype [S1, unofficial]. The method was developed to generate volumetric data in 3D polar coordinates $(r, \theta, z)$ for display on this hardware [S1, unofficial]. It was chosen over multi-view stereoscopy because multi-view does not easily support dynamic boolean modifiers per camera, and manually adding 24 cameras is cumbersome [S1, unofficial].

## Sources
[S1] Tiny volumetric display, https://mitxela.com/projects/candle, fetched 2026-10-26 (unofficial)
