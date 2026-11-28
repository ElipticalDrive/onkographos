# BlenderNeRF Data Extraction

Also called: COS, SOF, TTC

## Overview
BlenderNeRF Data Extraction is a method for generating synthetic training and testing datasets for Neural Radiance Fields (NeRF) and 3D Gaussian Splatting directly from a Blender scene. It solves the problem of manually exporting camera parameters and rendered images by automating the creation of `transforms_train.json` and `transforms_test.json` files alongside rendered image sequences. It distinguishes itself from general ray generation methods by providing three specific camera trajectory strategies—Subset of Frames (SOF), Train and Test Cameras (TTC), and Camera on Sphere (COS)—tailored for different novel view synthesis scenarios, while also supporting direct export of Gaussian Splatting point clouds.

## How it works
**Inputs:**
*   A Blender scene containing a 3D model and at least one camera.
*   Scene properties: `scene.render.resolution_percentage`, `scene.render.resolution_x`, `scene.render.resolution_y`, `scene.render.pixel_aspect_x`, `scene.render.pixel_aspect_y`, `scene.frame_start`, `scene.frame_end`, `scene.frame_step`, `scene.aabb`.
*   Method-specific properties: `scene.sof_dataset_name`, `scene.ttc_dataset_name`, `scene.cos_dataset_name`, `scene.train_frame_steps`, `scene.cos_nb_frames`, `scene.ttc_nb_frames`, `scene.splats`, `scene.splats_test_dummy`, `scene.render_frames`, `scene.logs`, `scene.test_data`, `scene.train_data`, `scene.nerf`.
*   For COS: `scene.cos_location`, `scene.cos_rotation`, `scene.cos_scale`, `scene.cos_radius`, `scene.cos_lens`, `scene.cos_seed`, `scene.cos_upper_views`, `scene.cos_outwards`.

**Outputs:**
*   `transforms_train.json`: List of frame dictionaries with `file_path` and `transform_matrix`.
*   `transforms_test.json`: List of frame dictionaries with `file_path` and `transform_matrix`.
*   Rendered images (if `scene.render_frames` is true).
*   `points3d.ply`: ASCII PLY file with vertex colors and normals (if `scene.splats` is true).
*   Log file (if `scene.logs` is true).

**Data Structures:**
*   `camera_intr_dict`: Dictionary with keys: `camera_angle_x` (float), `camera_angle_y` (float), `fl_x` (float), `fl_y` (float), `k1` (0.0), `k2` (0.0), `p1` (0.0), `p2` (0.0), `cx` (float), `cy` (float), `w` (float), `h` (float), `aabb_scale` (float).
*   `frame_data`: Dictionary with keys: `file_path` (string), `transform_matrix` (list of lists of floats, 4x4).
*   `camera_extr_dict`: List of `frame_data` dictionaries.

**Steps:**

1.  **Validation:** Check if a camera is selected. If not, abort with error. Assert camera type is PERSP for SOF and COS. Assert both train and test cameras are PERSP for TTC.
2.  **Intrinsics Calculation:** Call `get_camera_intrinsics` to populate `camera_intr_dict`.
    *   Calculate scale: $s = \text{scene.render.resolution\_percentage} / 100.0$.
    *   Calculate width: $w_{px} = \text{scene.render.resolution\_x} \times s$.
    *   Calculate height: $h_{px} = \text{scene.render.resolution\_y} \times s$.
    *   Calculate optical center: $c_x = w_{px} / 2.0$, $c_y = h_{px} / 2.0$.
    *   Calculate pixel aspect ratio: $p_{ar} = \text{scene.render.pixel\_aspect\_x} / \text{scene.render.pixel\_aspect\_y}$.
    *   If sensor fit is vertical and $w_{px} \le h_{px}$:
        *   Swap angles: $\theta_x = \text{camera\_angle\_y}$, $\theta_y = \text{camera\_angle\_x}$.
        *   Calculate focal lengths: $f_x = w_{px} / (2 \times \tan(\theta_x / 2))$, $f_y = f_x \times p_{ar}$.
    *   Else:
        *   $\theta_x = \text{camera\_angle\_x}$, $\theta_y = \text{camera\_angle\_y}$.
        *   $f_x = w_{px} / (2 \times \tan(\theta_x / 2))$, $f_y = h_{px} / (2 \times \tan(\theta_y / 2))$.
    *   Set `camera_intr_dict['w'] = w_{px}`, `camera_intr_dict['h'] = h_{px}`, `camera_intr_dict['cx'] = c_x`, `camera_intr_dict['cy'] = c_y`, `camera_intr_dict['fl_x'] = f_x`, `camera_intr_dict['fl_y'] = f_y`, `camera_intr_dict['camera_angle_x'] = \theta_x`, `camera_intr_dict['camera_angle_y'] = \theta_y`, `camera_intr_dict['aabb_scale'] = \text{scene.aabb}`.
3.  **Directory Setup:** Clean output directory name (replace unsupported characters with `_`). Create output path.
4.  **Logging:** If `scene.logs` is true, save log file.
5.  **Splats Export:** If `scene.splats` is true:
    *   Call `save_splats_ply`.
    *   Create temporary vertex colors for all visible meshes if they don't exist.
    *   Switch to OBJECT mode.
    *   Deselect all objects.
    *   Select only visible meshes (check `is_object_visible` for each object: true if not hidden in render and not in a hidden user collection).
    *   Export PLY file with normals, no attributes, ASCII format.
    *   Remove temporary vertex colors.
    *   Restore initial object mode.
6.  **Extrinsics Generation:**
    *   If `scene.test_data` is true:
        *   Call `get_camera_extrinsics` in TEST mode.
        *   Save `transforms_test.json`.
    *   If `scene.train_data` is true:
        *   Call `get_camera_extrinsics` in TRAIN mode.
        *   Save `transforms_train.json`.

**`get_camera_extrinsics` (mode, method):**
1.  Determine start frame, step, and end frame based on mode and method.
    *   **SOF:**
        *   Train: Start = `scene.frame_start`, Step = `scene.train_frame_steps`, End = `scene.frame_end`.
        *   Test: Start = `scene.frame_start`, Step = 1, End = `scene.frame_end`.
    *   **TTC:**
        *   Train: Start = `scene.frame_start`, Step = 1, End = `scene.frame_start + scene.ttc_nb_frames - 1`.
        *   Test: Start = `scene.frame_start`, Step = 1, End = `scene.frame_start + scene.ttc_nb_frames - 1` (or specific test camera frames).
    *   **COS:**
        *   Train: Start = `scene.frame_start`, Step = 1, End = `scene.frame_start + scene.cos_nb_frames - 1`.
        *   Test: Start = `scene.frame_start`, Step = 1, End = `scene.frame_end` (or specific test camera frames).
2.  Initialize `camera_extr_dict` as empty list.
3.  Loop $i$ from start to end with step:
    *   Set scene frame to $i$.
    *   Get filename for current frame (e.g., `frame_0001.png`).
    *   Determine file directory (train or test).
    *   Get camera transform matrix $M$ from Blender.
    *   Create `frame_data = {'file_path': path, 'transform_matrix': M}`.
    *   Append `frame_data` to `camera_extr_dict`.
4.  Restore initial frame.
5.  Return `camera_extr_dict`.

**`save_splats_ply`:**
1.  Iterate over all objects in scene.
2.  If `is_object_visible(obj)` is true:
    *   If no vertex color attribute exists, create one and set to black (0,0,0).
    *   Store original visibility state.
    *   Ensure object is visible for export.
3.  Export selected visible meshes to `points3d.ply` in ASCII format, including normals and vertex colors.
4.  Restore original visibility states.

**`is_object_visible(obj)`:**
1.  If `obj.hide_render` is true, return false.
2.  For each collection in `obj.users_collection`:
    *   If `collection.hide_render` is true, return false.
3.  Return true.

**Edge Cases and Pitfalls:**
*   No camera selected: Error.
*   Non-perspective cameras: Error for SOF/COS/TTC.
*   Splats test dummy: If `scene.splats_test_dummy` is true, return empty list for test transforms.
*   AABB: Must be integer power of 2.
*   Focal length: Avoid extreme values (30-70mm recommended). Avoid changing focal length during animation.
*   Sensor fit: Vertical fit with $w \le h$ requires angle swap.

## Subtopics

### Subset of Frames (SOF)
Renders every N frames from a camera animation for training, and all frames for testing. Best for interpolating or rendering large animations of static scenes. [SOF](52-sof.md)

### Train and Test Cameras (TTC)
Uses two separate user-defined cameras for training and testing. Best for evaluating NeRF on a specific test viewpoint. [TTC](52-ttc.md)

### Camera on Sphere (COS)
Uniformly samples random camera views directed at the center from a user-controlled sphere for training, and uses a selected camera for testing. Best for novel view synthesis from arbitrary viewpoints. [COS](52-cos.md)

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of frames rendered. Intrinsics calculation is constant time. PLY export is linear with respect to the number of visible vertices.
*   **Memory Complexity:** Linear with respect to the number of frames stored in JSON and the number of vertices in the PLY file.
*   **Performance:** Depends on Blender's rendering engine (Cycles/Eevee) if `scene.render_frames` is true. JSON/PLY generation is fast.
*   **Strengths:** Easy to use, integrates directly into Blender, supports both NeRF and Gaussian Splatting formats, provides multiple camera trajectory options.
*   **Limitations:** Requires Blender, synthetic data only, limited to perspective cameras, AABB must be power of 2, focal length should be constant.
*   **When to use:** When creating synthetic datasets for NeRF/Gaussian Splatting from Blender scenes.
*   **When not to use:** For real-world data, non-perspective cameras, or when high-performance rendering is not required (use simpler export scripts).

## Background
BlenderNeRF is an unofficial Blender add-on developed by Maximeraafat. It was created to simplify the process of generating synthetic datasets for NeRF and Gaussian Splatting. It builds on the NeRF format conventions and extends them with Instant NGP parameters. The add-on supports three main methods: SOF, TTC, and COS. It is designed for VFX artists, researchers, and graphics amateurs.

## Sources
[S1] maximeraafat/BlenderNeRF, https://github.com/maximeraafat/BlenderNeRF, fetched 2026-10-26 (unofficial)
