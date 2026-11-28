# Stencil Clip Mask Generation

Also called: ClipMaskOperation, RenderToClipMask

## Overview
Stencil Clip Mask Generation creates a stencil buffer mask for user interface clipping by applying a sequence of stencil operations based on a requested `ClipMaskOperation` (Set, SetInverse, or Intersect) [S1, unofficial]. It differs from standard depth-based clipping by allowing arbitrary closed surface geometries to define clipping regions via the stencil buffer, enabling complex UI overlays and split-screen effects [S1, unofficial].

## How it works
**Inputs:**
- A compiled geometry object stored in `this->geometries`.
- A creation shader (must be non-null).
- A translation vector (`translation.x`, `translation.y`).
- A projection matrix and a transform matrix.
- A `ClipMaskOperation` enum value (Set, SetInverse, Intersect).
- A scissor region (optional).

**Outputs:**
- Updated stencil buffer contents defining the clipping mask.

**Data Structures:**
- `BgfxFrameBuffer`: Contains `width` (int), `height` (int), `color` (shared_ptr<gfx::Texture>), `depth_stencil` (shared_ptr<gfx::Texture>), and `fb` (shared_ptr<gfx::FrameBuffer> or equivalent handle) [S1, unofficial].
- `stencilValue`: A monotonically increasing integer tracking the current stencil reference value [S1, unofficial].
- `stencilRef`: The current stencil reference value used in `BGFX_STENCIL_FUNC_REF`, stored as a class member [S1, unofficial].

**Steps:**
1. Locate the compiled geometry in `this->geometries`. If not found or if the creation shader is null, return immediately [S1, unofficial].
2. Compute the Model-View-Projection (MVP) matrix:
   - Create a translation matrix: `translate_mat = Rml::Matrix4f::Translate(translation.x, translation.y, 0.0f)`.
   - Compute `mvp = projection * transform * translate_mat` [S1, unofficial].
3. Convert the `mvp` matrix to a 16-element float array and set it as the `u_transform` uniform [S1, unofficial].
4. Apply stencil operations based on `ClipMaskOperation`:
   - **Set**: Increment `stencilValue`, assign it to `stencilRef`. Configure stencil to always pass, write the reference value, and zero on stencil/depth fail [S1, unofficial].
   - **SetInverse**: First, draw a fullscreen quad with an identity transform to fill the stencil with `stencilRef` using REPLACE operations. Then, draw the geometry with REPLACE operations and stencil reference 0 [S1, unofficial].
   - **Intersect**: Save `old_ref = stencilRef`. Increment `stencilValue`. If `stencilRef == old_ref + 1` (sequential), use EQUAL test with INCR operation; otherwise, use LEQUAL test with REPLACE operation [S1, unofficial].
5. Re-apply the scissor region if enabled, set the final stencil state, and submit the draw call [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_clip_mask(geometries, shader, translation, projection, transform, operation, scissor_enabled, scissor_region, stencil_value, stencil_ref):
    # Step 1: Locate geometry
    geometry = geometries.get(translation) # Assuming translation or ID maps to geometry
    if geometry is None or shader is None:
        return

    # Step 2: Compute MVP
    translate_mat = Rml.Matrix4f.translate(translation.x, translation.y, 0.0f)
    mvp = projection * transform * translate_mat

    # Step 3: Set uniform
    mvp_array = mvp.to_float_array_16()
    shader.set_uniform("u_transform", mvp_array)

    # Step 4: Apply stencil operations
    if operation == "Set":
        stencil_value += 1
        stencil_ref = stencil_value
        # Configure stencil: always pass, write ref, zero on fail
        set_stencil_func(BGFX_STENCIL_FUNC_ALWAYS, stencil_ref, 0xFF)
        set_stencil_op(BGFX_STENCIL_OP_KEEP, BGFX_STENCIL_OP_KEEP, BGFX_STENCIL_OP_REPLACE)
        set_depth_func(BGFX_DEPTH_FUNC_ALWAYS) # Assuming depth is handled via stencil zeroing on fail

    elif operation == "SetInverse":
        # Draw fullscreen quad with identity transform
        identity_mat = Rml.Matrix4f.identity()
        shader.set_uniform("u_transform", identity_mat.to_float_array_16())
        set_stencil_func(BGFX_STENCIL_FUNC_ALWAYS, stencil_ref, 0xFF)
        set_stencil_op(BGFX_STENCIL_OP_REPLACE, BGFX_STENCIL_OP_REPLACE, BGFX_STENCIL_OP_REPLACE)
        draw_fullscreen_quad()

        # Draw geometry with stencil ref 0
        shader.set_uniform("u_transform", mvp_array)
        set_stencil_func(BGFX_STENCIL_FUNC_ALWAYS, 0, 0xFF)
        set_stencil_op(BGFX_STENCIL_OP_REPLACE, BGFX_STENCIL_OP_REPLACE, BGFX_STENCIL_OP_REPLACE)
        draw_geometry(geometry)

    elif operation == "Intersect":
        old_ref = stencil_ref
        stencil_value += 1
        if stencil_ref == old_ref + 1:
            set_stencil_func(BGFX_STENCIL_FUNC_EQUAL, stencil_ref, 0xFF)
            set_stencil_op(BGFX_STENCIL_OP_KEEP, BGFX_STENCIL_OP_KEEP, BGFX_STENCIL_OP_INCR)
        else:
            set_stencil_func(BGFX_STENCIL_FUNC_LEQUAL, stencil_ref, 0xFF)
            set_stencil_op(BGFX_STENCIL_OP_REPLACE, BGFX_STENCIL_OP_REPLACE, BGFX_STENCIL_OP_REPLACE)

    # Step 5: Finalize
    if scissor_enabled:
        set_scissor_region(scissor_region)
    set_final_stencil_state()
    submit_draw_call(geometry)
```

**Parameters:**
- `stencilValue`: Monotonically increasing integer tracking the current stencil reference value. Incremented on each Set or Intersect operation [S1, unofficial].
- `stencilRef`: Current stencil reference value used in `BGFX_STENCIL_FUNC_REF`. Stored as a class member [S1, unofficial].

**Edge Cases and Pitfalls:**
- If the geometry is not found or the shader is null, the function returns immediately without modifying the stencil buffer [S1, unofficial].
- Precision issues may arise if `stencilValue` overflows, though this is unlikely given typical UI clipping scenarios [G].
- The `Intersect` operation relies on sequential increments; if the sequence is broken, it falls back to a less restrictive LEQUAL test [S1, unofficial].

## Operating characteristics
- **Time Complexity:** O(1) per clipping operation, assuming constant-time matrix multiplication and stencil buffer updates.
- **Memory Complexity:** O(1) additional memory, as it relies on the existing stencil buffer and frame buffer structures [S1, unofficial].
- **Performance:** Performance depends on the complexity of the geometry and the number of clipping operations. Fullscreen quad draws for `SetInverse` may be costly if done frequently [G].
- **Strengths:** Allows for complex, arbitrary clipping shapes using standard stencil buffer operations. Efficient for UI clipping where the clipping region is static or changes infrequently [S1, unofficial].
- **Limitations:** Limited by the stencil buffer width (typically 8 bits, allowing up to 256 levels of nesting). Not suitable for dynamic, high-frequency clipping updates [G].
- **When to use:** Ideal for UI clipping, split-screen effects, and other scenarios requiring precise, arbitrary clipping regions [S1, unofficial].
- **When not to use:** Not suitable for volumetric data clipping where depth-based methods are more appropriate, or for dynamic scenes with frequent clipping changes [G].

## Background
Stencil Clip Mask Generation is part of the Damascene engine's UI rendering system, specifically designed to handle clipping for UI elements using the BGFX graphics library [S1, unofficial]. It leverages the stencil buffer to create masks that can be used to clip subsequent rendering passes, enabling features like split-screen and custom clipping regions [S1, unofficial].

## Sources
[S1] CDevJoud/Damascene, https://github.com/CDevJoud/Damascene, fetched 2026-10-27 (unofficial)
