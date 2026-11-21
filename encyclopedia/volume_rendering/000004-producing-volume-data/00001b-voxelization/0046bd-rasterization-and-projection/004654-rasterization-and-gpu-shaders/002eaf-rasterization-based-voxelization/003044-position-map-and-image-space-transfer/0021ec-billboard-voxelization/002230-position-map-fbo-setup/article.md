# Position Map FBO Setup

Also called: initPositionFBO

## Overview
Creates a Framebuffer Object (FBO) with a color attachment (position map) and a depth attachment for voxelization rendering [S1, unofficial]. It prepares the GPU state for the first pass of billboard voxelization, where instanced quads are rendered to initialize voxel positions in a 3D texture [S1, unofficial].

## How it works
**Inputs and outputs:**
- Input: None (setup routine).
- Output: A bound FBO with attached textures/renderbuffers ready for rendering.

**Data structures:**
1. **Position Map Texture (Color Attachment):**
   - Type: 2D Texture.
   - Internal Format: `GL_RGBA32F` [S1, unofficial].
   - Dimensions: Width and height matching the window dimensions [S1, unofficial].
   - Filtering: Nearest (`GL_NEAREST`) [S1, unofficial].
   - Wrapping: Clamp to Edge (`GL_CLAMP_TO_EDGE`) [S1, unofficial].
2. **Depth Buffer:**
   - Type: Renderbuffer [S1, unofficial].
   - Format: `GL_DEPTH_COMPONENT` [S1, unofficial].
   - Dimensions: Width and height matching the window dimensions [S1, unofficial].

**Steps:**
1. Generate a Framebuffer Object ID.
2. Bind the FBO.
3. Generate a 2D texture object.
4. Bind the 2D texture.
5. Allocate storage for the texture using `glTexImage2D` or `glTexStorage2D` with format `GL_RGBA32F`, dimensions equal to window width/height, and `GL_FLOAT` type.
6. Set texture parameters:
   - `GL_TEXTURE_MIN_FILTER` to `GL_NEAREST`.
   - `GL_TEXTURE_MAG_FILTER` to `GL_NEAREST`.
   - `GL_TEXTURE_WRAP_S` to `GL_CLAMP_TO_EDGE`.
   - `GL_TEXTURE_WRAP_T` to `GL_CLAMP_TO_EDGE`.
7. Attach the texture to the FBO at `GL_COLOR_ATTACHMENT0`.
8. Generate a renderbuffer object.
9. Bind the renderbuffer.
10. Allocate storage for the renderbuffer using `glRenderbufferStorage` with format `GL_DEPTH_COMPONENT`, dimensions equal to window width/height.
11. Attach the renderbuffer to the FBO at `GL_DEPTH_ATTACHMENT`.
12. Verify framebuffer completeness using `glCheckFramebufferStatus`.
13. Unbind the FBO.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def setup_position_map_fbo(window_width, window_height):
    # 1. Generate FBO
    fbo_id = glGenFramebuffers(1)
    glBindFramebuffer(GL_FRAMEBUFFER, fbo_id)

    # 2. Create and configure Position Map Texture (Color Attachment)
    pos_map_tex = glGenTextures(1)
    glBindTexture(GL_TEXTURE_2D, pos_map_tex)
    
    # Allocate RGBA32F texture
    glTexImage2D(
        GL_TEXTURE_2D, 0, GL_RGBA32F, 
        window_width, window_height, 0, 
        GL_RGBA, GL_FLOAT, None
    )
    
    # Set filtering and wrapping
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_NEAREST)
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_NEAREST)
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE)
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE)
    
    # Attach to FBO
    glFramebufferTexture2D(
        GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, 
        GL_TEXTURE_2D, pos_map_tex, 0
    )

    # 3. Create and configure Depth Renderbuffer
    depth_rb = glGenRenderbuffers(1)
    glBindRenderbuffer(GL_RENDERBUFFER, depth_rb)
    
    # Allocate depth buffer
    glRenderbufferStorage(
        GL_RENDERBUFFER, GL_DEPTH_COMPONENT, 
        window_width, window_height
    )
    
    # Attach to FBO
    glFramebufferRenderbuffer(
        GL_FRAMEBUFFER, GL_DEPTH_ATTACHMENT, 
        GL_RENDERBUFFER, depth_rb
    )

    # 4. Check completeness
    status = glCheckFramebufferStatus(GL_FRAMEBUFFER)
    if status != GL_FRAMEBUFFER_COMPLETE:
        raise RuntimeError(f"Framebuffer incomplete: {status}")

    # 5. Cleanup bindings
    glBindRenderbuffer(GL_RENDERBUFFER, 0)
    glBindTexture(GL_TEXTURE_2D, 0)
    glBindFramebuffer(GL_FRAMEBUFFER, 0)

    return fbo_id, pos_map_tex, depth_rb
```

**Parameters:**
- `window_width`: Integer, typically the application window width [S1, unofficial].
- `window_height`: Integer, typically the application window height [S1, unofficial].

**Edge cases and pitfalls:**
- If `window_width` or `window_height` is 0, texture allocation may fail or produce undefined behavior [G].
- Ensure the OpenGL context is current before calling GL functions [G].
- The depth buffer format `GL_DEPTH_COMPONENT` may require specific driver support; `GL_DEPTH_COMPONENT24` or `GL_DEPTH_COMPONENT32` might be needed for higher precision, though the source specifies `GL_DEPTH_COMPONENT` [S1, unofficial].

## Subtopics

### Voxelization Pass
A subsequent pass that renders the scene into a 3D texture (`tex3D`) using the position map FBO setup. It sets the viewport to `tex_size` x `tex_size`, binds the default framebuffer, clears buffers, disables face culling and depth testing, binds `tex3D` as a write-only image texture at unit 0 with `GL_RGBA8` format, activates the voxelization shader, sets uniforms `voxelImage` to 0 and `sceneScale` to `scene_scale`, renders the scene using an orthographic camera with frustum `(-1, 1, -1, 1, 1, -1)`, and inserts a memory barrier for texture fetches while generating mipmaps [S2, unofficial].

## Operating characteristics
Not given in the sources.

## Background
Originated in the context of GPU-based voxelization techniques for volume rendering, specifically within the Cloud-Renderer project [S1, unofficial]. It is part of a two-pass voxelization approach where the first pass initializes voxel positions using billboards [S1, unofficial].

## Sources
[S1] jaafersheriff/Cloud-Renderer, https://github.com/jaafersheriff/Cloud-Renderer, fetched 2026-10-27 (unofficial)
[S2] kbladin/Voxel_Cone_Tracing, https://github.com/kbladin/Voxel_Cone_Tracing, fetched 2026-10-26 (unofficial)
