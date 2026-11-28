# FBO Texture & Renderbuffer Generation

Also called: Dynamic FBO Allocation, Framebuffer Attachment Setup

## Overview
Allocates and attaches 2D textures and depth renderbuffers to Framebuffer Objects (FBOs), handling cleanup and dynamic viewport resizing [S1, unofficial]. It sets up the color attachment (position map) and depth attachment required for rasterization-based voxelization [S1, unofficial]. Unlike static allocation, this method dynamically adjusts texture dimensions to match the current viewport, ensuring the position map resolution matches the rendering target [S1, unofficial].

## How it works
**Inputs and outputs:**
- `viewport_width`, `viewport_height`: Integer dimensions of the current viewport.
- `textureID`: Pointer to the existing color texture ID.
- `depthBuffer`: Pointer to the existing depth renderbuffer ID.
- `attachmentID`: The attachment point index (e.g., `GL_COLOR_ATTACHMENT0`).

**Data structures:**
- OpenGL Texture Object: 2D texture, format `GL_RGBA16F`, dimensions `viewport_width` x `viewport_height`.
- OpenGL Renderbuffer Object: Depth buffer, format `GL_DEPTH_COMPONENT`, dimensions `viewport_width` x `viewport_height`.

**Steps:**
1. **Depth Buffer Management:**
   - If `depthBuffer` is non-zero, delete the existing renderbuffer using `glDeleteRenderbuffers`.
   - Generate a new renderbuffer ID.
   - Bind the renderbuffer.
   - Allocate storage using `glRenderbufferStorage` with format `GL_DEPTH_COMPONENT` and dimensions `viewport_width` x `viewport_height`.

2. **Color Texture Management:**
   - If `textureID` is non-zero:
     - Bind the texture target to `0` to avoid OpenGL errors during deletion [S1, unofficial].
     - Delete the existing texture using `glDeleteTextures`.
   - Generate a new texture ID.
   - Bind the texture to `GL_TEXTURE_2D`.
   - Allocate storage using `glTexImage2D` with:
     - Internal format: `GL_RGBA16F` (16-bit floating point per channel) [S1, unofficial].
     - Width: `viewport_width`.
     - Height: `viewport_height`.
     - Format: `GL_RGBA`.
     - Type: `GL_FLOAT`.
     - Data: `NULL` (uninitialized).
   - Set texture parameters:
     - `GL_TEXTURE_MIN_FILTER`: `GL_LINEAR` [S1, unofficial].
     - `GL_TEXTURE_MAG_FILTER`: `GL_LINEAR` [S1, unofficial].
     - `GL_TEXTURE_WRAP_S`: `GL_CLAMP_TO_EDGE` [S1, unofficial].
     - `GL_TEXTURE_WRAP_T`: `GL_CLAMP_TO_EDGE` [S1, unofficial].
   - Unbind the texture (bind to `0`).

3. **Framebuffer Attachment:**
   - Bind the target FBO.
   - Attach the new texture to the FBO at `attachmentID` using `glFramebufferTexture2D` with target `GL_TEXTURE_2D` and level `0`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_fbo_attachments(viewport_width, viewport_height, textureID, depthBuffer, attachmentID):
    # 1. Depth Buffer Management
    if depthBuffer != 0:
        glDeleteRenderbuffers(1, [depthBuffer])
    
    new_depth_buffer = glGenRenderbuffers(1)
    glBindRenderbuffer(GL_RENDERBUFFER, new_depth_buffer)
    glRenderbufferStorage(GL_RENDERBUFFER, GL_DEPTH_COMPONENT, viewport_width, viewport_height)
    
    # 2. Color Texture Management
    if textureID != 0:
        glBindTexture(GL_TEXTURE_2D, 0) # Unbind before delete to avoid errors [S1, unofficial]
        glDeleteTextures(1, [textureID])
        
    new_texture = glGenTextures(1)
    glBindTexture(GL_TEXTURE_2D, new_texture)
    
    # Allocate RGBA16F texture
    glTexImage2D(
        GL_TEXTURE_2D, 
        0, 
        GL_RGBA16F, 
        viewport_width, 
        viewport_height, 
        0, 
        GL_RGBA, 
        GL_FLOAT, 
        None
    )
    
    # Set filters and wrap mode
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR)
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR)
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE)
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE)
    
    glBindTexture(GL_TEXTURE_2D, 0) # Unbind
    
    # 3. Framebuffer Attachment
    # Assumes current bound framebuffer is the target FBO
    glFramebufferTexture2D(
        GL_FRAMEBUFFER, 
        attachmentID, 
        GL_TEXTURE_2D, 
        new_texture, 
        0
    )
    
    return new_texture, new_depth_buffer
```

**Parameters:**
- `viewport_width`, `viewport_height`: Integer dimensions used for renderbuffer and texture allocation, updated on resize [S1, unofficial].

**Edge cases and pitfalls:**
- Textures must be bound to target `0` before deletion to avoid OpenGL errors [S1, unofficial].
- The internal format `GL_RGBA16F` provides sufficient precision for position map storage but requires hardware support for floating-point framebuffers.

## Operating characteristics
- **Time Complexity:** O(1) per frame resize, as it involves a fixed number of OpenGL calls.
- **Memory Complexity:** O(W * H) for the texture and renderbuffer, where W and H are viewport dimensions.
- **Strengths:** Dynamic resizing ensures the position map resolution matches the rendering target, avoiding aliasing or wasted memory.
- **Limitations:** Requires OpenGL 3.0+ for core FBO functionality and floating-point texture support. Frequent resizing can cause performance hiccups if not throttled.

## Background
Originated in OpenGL 3.0 with the introduction of Framebuffer Objects (FBOs) as a replacement for the fixed-function framebuffer. This method is commonly used in rasterization-based voxelization pipelines where position maps are rendered to GPU memory for subsequent volume processing [S1, unofficial].

## Sources
[S1] IVSOP/VoxelEngine, https://github.com/IVSOP/VoxelEngine, fetched 2026-10-26 (unofficial)
