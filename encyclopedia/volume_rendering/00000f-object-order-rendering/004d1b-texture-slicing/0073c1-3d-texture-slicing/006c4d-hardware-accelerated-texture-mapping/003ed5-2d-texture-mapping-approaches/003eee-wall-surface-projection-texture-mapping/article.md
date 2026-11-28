# Wall Surface Projection & Texture Mapping

Also called: Axis-aligned Texture Projection, Planar Mapping, renderBlockSurfaces

## Overview
Wall Surface Projection & Texture Mapping projects a 3D block face to 2D screen coordinates, handles X/Y clipping, calculates UV mapping with perspective-correct interpolation, applies per-face shading, and draws pixels [S1, unofficial]. It operates on discrete axis-aligned faces rather than volumetric samples, making it suited for block-based environments where shading is per-face [S1, unofficial].

## How it works
**Inputs:**
- A 3D voxel grid (chunk) where each block has a state (active/inactive) and a texture index [S1, unofficial].
- Camera position and orientation (player position `x, y, z` and angles `angle, lookAngle`) [S1, unofficial].
- Screen dimensions (`SCR_WIDTH`, `SCR_HEIGHT`) [S1, unofficial].
- Texture data: a 1D array `textureMap` of RGB values, with dimensions `width` and `height` [S1, unofficial].

**Outputs:**
- A 2D screen buffer with pixels drawn according to the projected textures [S1, unofficial].

**Data Structures:**
- `Block`: Contains `visibleFaces` (boolean flags for 6 faces: XPos, XNeg, YPos, YNeg, ZPos, ZNeg) and `textureIndex` [S1, unofficial].
- `Texture`: Contains `width`, `height`, and `textureMap` (array of size `width * height * 3`) [S1, unofficial].
- `Chunk`: A 3D array of `Block` objects of size `CHUNK_SIZE x CHUNK_SIZE x CHUNK_SIZE` [S1, unofficial].

**Steps:**
1. Compute `horizontalStep = (texture.width * uvU) / (xPos2 - xPos1)` [S1, unofficial].
2. Clamp `xPos1` and `xPos2` to `[0, SCR_WIDTH]`, adjusting `horizontalStart` [S1, unofficial].
3. Compute `distX = max(1, xPos2 - xPos1)` [S1, unofficial].
4. For each `x`, compute `y1 = bottomPos1 + distBottom * (x - originalX1 + 0.5) / distX` and `y2 = topPos1 + distTop * (x - originalX1 + 0.5) / distX` [S1, unofficial].
5. Compute `verticalStep = (texture.height * uvV) / (y2 - y1)` [S1, unofficial].
6. Clamp `y1` and `y2` to `[0, SCR_HEIGHT]`, adjusting `verticalStart` [S1, unofficial].
7. For each `y`, compute texture index `p` using modulo wrapping on `height`/`width` [S1, unofficial].
8. Apply shading: `max(textureMap[p+c] - shade/2, 0)` [S1, unofficial].
9. Call `drawPixel(x, y, color, distanceToPlayer)` [S1, unofficial].
10. Increment `verticalStart` and `horizontalStart` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class RgbColor:
    def __init__(self, r, g, b):
        self.r, self.g, self.b = r, g, b

class Screen:
    def draw_pixel(self, x, y, color, distance):
        pass

class FaceCorners:
    def __init__(self):
        self.x1, self.y1, self.z1 = 0.0, 0.0, 0.0
        self.x2, self.y2, self.z2 = 0.0, 0.0, 0.0
        self.bottom_y, self.dist_bottom = 0.0, 0.0
        self.top_y, self.dist_top = 0.0, 0.0
        self.original_x1 = 0.0

def is_face_visible(chunk, x, y, z, face_idx):
    offsets = [(1,0,0), (-1,0,0), (0,1,0), (0,-1,0), (0,0,1), (0,0,-1)]
    dx, dy, dz = offsets[face_idx]
    nx, ny, nz = x + dx, y + dy, z + dz
    size = len(chunk.blocks)
    return not (0 <= nx < size and 0 <= ny < size and 0 <= nz < size and chunk.blocks[nx][ny][nz].is_active())

def clip_behind_player(corners):
    y1, y2 = corners.y1, corners.y2
    if y1 < 1 and y2 < 1:
        return
    if y1 < 1:
        t = (1 - y1) / (y2 - y1)
        corners.x1 += t * (corners.x2 - corners.x1)
        corners.y1 = 1
        corners.z1 += t * (corners.z2 - corners.z1)
    elif y2 < 1:
        t = (1 - y2) / (y1 - y2)
        corners.x2 += t * (corners.x1 - corners.x2)
        corners.y2 = 1
        corners.z2 += t * (corners.z1 - corners.z1)

def render_wall_surface_projection(chunk, camera, screen_width, screen_height, textures, shade):
    CHUNK_SIZE = len(chunk.blocks)
    for x in range(CHUNK_SIZE):
        for y in range(CHUNK_SIZE):
            for z in range(CHUNK_SIZE):
                if not chunk.blocks[x][y][z].is_active():
                    continue
                block = chunk.blocks[x][y][z]
                visible_faces = [f for f in range(6) if is_face_visible(chunk, x, y, z, f)]
                for face in visible_faces:
                    corners = project_face_to_screen(face)
                    clip_behind_player(corners)
                    dist_x = max(1, corners.x2 - corners.x1)
                    horizontal_step = (textures[block.texture_index].width * face.uv_u) / dist_x
                    x1_clamped = max(0, min(screen_width, corners.x1))
                    x2_clamped = max(0, min(screen_width, corners.x2))
                    horizontal_start = horizontal_step * x1_clamped
                    for x in range(int(x1_clamped), int(x2_clamped)):
                        y1 = corners.bottom_y + corners.dist_bottom * (x - corners.original_x1 + 0.5) / dist_x
                        y2 = corners.top_y + corners.dist_top * (x - corners.original_x1 + 0.5) / dist_x
                        dist_y = max(1, y2 - y1)
                        vertical_step = (textures[block.texture_index].height * face.uv_v) / dist_y
                        y1_clamped = max(0, min(screen_height, y1))
                        y2_clamped = max(0, min(screen_height, y2))
                        vertical_start = vertical_step * y1_clamped
                        for y in range(int(y1_clamped), int(y2_clamped)):
                            tex = textures[block.texture_index]
                            p = ((tex.height - (int(vertical_start) % tex.height) - 1) * 3 * tex.width +
                                 (int(horizontal_start) % tex.width) * 3)
                            r = max(tex.texture_map[p + 0] - shade / 2, 0)
                            g = max(tex.texture_map[p + 1] - shade / 2, 0)
                            b = max(tex.texture_map[p + 2] - shade / 2, 0)
                            screen.draw_pixel(x, y, RgbColor(r, g, b), camera.distance_to_face)
                            vertical_start += vertical_step
                        horizontal_start += horizontal_step
```

**Parameters:**
- `CHUNK_SIZE`: Size of the voxel chunk. Not given in the sources.
- `shade`: Shading intensity. Not given in the sources.
- `uvU`, `uvV`: UV scaling factors. Not given in the sources.

**Edge Cases and Pitfalls:**
- **Division by Zero:** If `xPos2 - xPos1` is 0, `distX` is forced to 1 to prevent division by zero [S1, unofficial].
- **Clipping:** Faces behind the player must be clipped to avoid rendering artifacts. The source uses a check against the near plane `y=1` using linear interpolation [S1, unofficial].
- **Shading:** The source code applies shading using `max(textureMap[p+c] - shade/2, 0)` for each channel [S1, unofficial].

## Subtopics

### Face Transform
Maps a whole face by returning an SVG matrix that maps the unit square $[0,1]^2$ onto the face's projected quad, allowing gradients or patterns defined in $0$–$1$ space to follow the face [S2, unofficial]. It covers radial gradients, diagonal axes, and patterns which endpoint projection cannot handle [S2, unofficial]. The mapping uses the quad's corners 0, 1, and 3 as origin, $+U$, and $+V$ respectively to perform an affine (parallelogram) fit [S2, unofficial]. To float content off the surface, pass a second argument representing a 3D lift vector (e.g., face normal times a distance) before projecting, ensuring correct foreshortening under perspective [S2, unofficial]. Under perspective projection, the affine fit is only a close approximation, not exact [S2, unofficial]. Code example: `const m = h.faceTransform(f, [f.n[0] * 0.2, f.n[1] * 0.2, f.n[2] * 0.2]);` [S2, unofficial].

## Operating characteristics
- **Time Complexity:** O(N) where N is the number of visible faces, with each face processed in time proportional to its screen area [G].
- **Memory Complexity:** O(V) where V is the number of voxels in the chunk, for storing the voxel grid, plus O(T) for texture memory [G].
- **Strengths:** Simple to implement for block-based environments and efficient for scenes with large flat surfaces [G].
- **Limitations:** Limited to axis-aligned faces and does not handle curved surfaces or non-axis-aligned geometry well [G]. Shading is per-face rather than per-voxel, which may appear blocky [G].
- **When to Use:** Block-based games or voxel editors where performance is critical and geometry is simple [G].
- **When Not to Use:** Scenes with complex geometry, curved surfaces, or high-fidelity shading requirements [G].

## Background
This method is derived from early 3D rendering techniques used in voxel-based games, such as those inspired by the 1993 era of PC gaming [S1, unofficial]. It represents a simplified approach to texture mapping in 3D space, focusing on axis-aligned planes [S1, unofficial].

## Sources
[S1] VidFurlan/Voxel1993, https://github.com/VidFurlan/Voxel1993, fetched 2026-10-27 (unofficial)
[S2] meodai/heerich, https://github.com/meodai/heerich, fetched 2026-10-26 (unofficial)
