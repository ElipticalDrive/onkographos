# Frustum-Box Containment Test

Also called: Frustum vs AABB containment, Inigo Quilez's frustum culling approach

## Overview
Determines if an axis-aligned bounding box (AABB) is fully contained within a view frustum defined by a projection * view matrix, returning true if the box is potentially visible and false if it is completely outside [S1, unofficial]. It differs from [AABB Culling](1789-aabb-culling.md) by testing mutual containment (box inside frustum AND frustum inside box) rather than just box outside frustum, and from [Bounding Box Culling](1806-bounding-box-culling.md) by operating on explicit frustum planes rather than generic bounding volumes [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A 4x4 projection-view matrix $M$ (column-major), and an AABB defined by minimum corner $(x_l, y_l, z_l)$ and maximum corner $(x_h, y_h, z_h)$ [S1, unofficial].
- Output: Boolean `true` if the box is not completely outside the frustum, `false` otherwise [S1, unofficial].

**Data Structures:**
- `Plane`: A 4-element vector $(n_x, n_y, n_z, d)$ representing the plane equation $n_x x + n_y y + n_z z + d = 0$ [S1, unofficial]. The normal $(n_x, n_y, n_z)$ is normalized [S1, unofficial].
- `Frustum`: Contains an array of 6 `Plane` objects and an array of 8 `vec4` points representing the frustum corners in world space [S1, unofficial].
- `Box`: A flat array of 6 floats `[x_l, y_l, z_l, x_h, y_h, z_h]` [S1, unofficial].

**Steps:**
1. **Extract Frustum Planes:** Compute the 6 planes from the matrix $M$ [S1, unofficial]. Let $R_i$ be the $i$-th column of $M$ (for $i=0,1,2,3$) [S1, unofficial]. The planes are:
   $$ P_{\pm x} = R_3 \pm R_0 $$
   $$ P_{\pm y} = R_3 \pm R_1 $$
   $$ P_{\pm z} = R_3 \pm R_2 $$
   Normalize each plane $P$ by computing $s = 1/\sqrt{n_x^2+n_y^2+n_z^2}$ and updating $P \leftarrow P \cdot s$ [S1, unofficial].
2. **Test Box Against Planes:** For each of the 6 planes, determine if all 8 corners of the box lie behind the plane [S1, unofficial]. A corner $C_j$ is behind the plane if $P \cdot C_j < 0.0$ [S1, unofficial]. If the count of corners behind the plane is 8, the box is outside the frustum [S1, unofficial].
3. **Test Frustum Against Box:** Compute the 8 corners of the frustum in world space (using the inverse projection-view matrix) [S1, unofficial]. For each axis ($x, y, z$), check if all 8 frustum corners lie outside the box on the same side (i.e., all $> \text{box}_{\text{max}}$ or all $< \text{box}_{\text{min}}$) [S1, unofficial]. If so, the frustum is outside the box, implying no intersection [S1, unofficial].
4. **Result:** If neither condition triggers, return `true` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def frustum_box_containment(proj_view_matrix, box_min, box_max):
    # Step 1: Extract and normalize planes from column-major matrix M[row][col]
    c0 = [proj_view_matrix[0][0], proj_view_matrix[1][0], proj_view_matrix[2][0], proj_view_matrix[3][0]]
    c1 = [proj_view_matrix[0][1], proj_view_matrix[1][1], proj_view_matrix[2][1], proj_view_matrix[3][1]]
    c2 = [proj_view_matrix[0][2], proj_view_matrix[1][2], proj_view_matrix[2][2], proj_view_matrix[3][2]]
    c3 = [proj_view_matrix[0][3], proj_view_matrix[1][3], proj_view_matrix[2][3], proj_view_matrix[3][3]]
    
    frustum_planes = [
        [c3[0]+c0[0], c3[1]+c0[1], c3[2]+c0[2], c3[3]+c0[3]],
        [c3[0]-c0[0], c3[1]-c0[1], c3[2]-c0[2], c3[3]-c0[3]],
        [c3[0]+c1[0], c3[1]+c1[1], c3[2]+c1[2], c3[3]+c1[3]],
        [c3[0]-c1[0], c3[1]-c1[1], c3[2]-c1[2], c3[3]-c1[3]],
        [c3[0]+c2[0], c3[1]+c2[1], c3[2]+c2[2], c3[3]+c2[3]],
        [c3[0]-c2[0], c3[1]-c2[1], c3[2]-c2[2], c3[3]-c2[3]]
    ]
    
    normalized_planes = []
    for p in frustum_planes:
        nx, ny, nz, d = p
        len_sq = nx*nx + ny*ny + nz*nz
        if len_sq == 0:
            continue # Guard against degenerate plane [G]
        s = 1.0 / (len_sq ** 0.5)
        normalized_planes.append([nx*s, ny*s, nz*s, d*s])
        
    # Step 2: Test box against planes
    corners = []
    for x in [box_min[0], box_max[0]]:
        for y in [box_min[1], box_max[1]]:
            for z in [box_min[2], box_max[2]]:
                corners.append([x, y, z, 1.0])
                
    for p in normalized_planes:
        out_count = 0
        for c in corners:
            dot = p[0]*c[0] + p[1]*c[1] + p[2]*c[2] + p[3]*c[3]
            if dot < 0.0:
                out_count += 1
        if out_count == 8:
            return False
            
    # Step 3: Test frustum against box
    frustum_corners = get_frustum_corners(proj_view_matrix)
    
    for i in range(3):
        all_max = all(frustum_corners[j][i] > box_max[i] for j in range(8))
        all_min = all(frustum_corners[j][i] < box_min[i] for j in range(8))
        if all_max or all_min:
            return False
            
    return True

def get_frustum_corners(proj_view_matrix):
    inv_matrix = invert_4x4(proj_view_matrix) # Standard matrix inversion [G]
    corners = []
    for x in [-1, 1]:
        for y in [-1, 1]:
            for z in [-1, 1]:
                v = [x, y, z, 1.0]
                res = [
                    inv_matrix[0][0]*v[0] + inv_matrix[1][0]*v[1] + inv_matrix[2][0]*v[2] + inv_matrix[3][0]*v[3],
                    inv_matrix[0][1]*v[0] + inv_matrix[1][1]*v[1] + inv_matrix[2][1]*v[2] + inv_matrix[3][1]*v[3],
                    inv_matrix[0][2]*v[0] + inv_matrix[1][2]*v[1] + inv_matrix[2][2]*v[2] + inv_matrix[3][2]*v[3],
                    inv_matrix[0][3]*v[0] + inv_matrix[1][3]*v[1] + inv_matrix[2][3]*v[2] + inv_matrix[3][3]*v[3]
                ]
                if res[3] != 0:
                    res[0] /= res[3]
                    res[1] /= res[3]
                    res[2] /= res[3]
                corners.append(res)
    return corners

def invert_4x4(m):
    inv = [[0]*4 for _ in range(4)]
    inv[0][0] = m[1][1]*m[2][2]*m[3][3] - m[1][1]*m[2][3]*m[3][2] - m[2][1]*m[1][2]*m[3][3] + m[2][1]*m[1][3]*m[3][2] + m[3][1]*m[1][2]*m[2][3] - m[3][1]*m[1][3]*m[2][2]
    inv[0][1] = -m[0][1]*m[2][2]*m[3][3] + m[0][1]*m[2][3]*m[3][2] + m[2][1]*m[0][2]*m[3][3] - m[2][1]*m[0][3]*m[3][2] - m[3][1]*m[0][2]*m[2][3] + m[3][1]*m[0][3]*m[2][2]
    inv[0][2] = m[0][1]*m[1][2]*m[3][3] - m[0][1]*m[1][3]*m[3][2] - m[1][1]*m[0][2]*m[3][3] + m[1][1]*m[0][3]*m[3][2] + m[3][1]*m[0][2]*m[1][3] - m[3][1]*m[0][3]*m[1][2]
    inv[0][3] = -m[0][1]*m[1][2]*m[2][3] + m[0][1]*m[1][3]*m[2][2] + m[1][1]*m[0][2]*m[2][3] - m[1][1]*m[0][3]*m[2][2] - m[2][1]*m[0][2]*m[1][3] + m[2][1]*m[0][3]*m[1][2]
    inv[1][0] = -m[1][0]*m[2][2]*m[3][3] + m[1][0]*m[2][3]*m[3][2] + m[2][0]*m[1][2]*m[3][3] - m[2][0]*m[1][3]*m[3][2] - m[3][0]*m[1][2]*m[2][3] + m[3][0]*m[1][3]*m[2][2]
    inv[1][1] = m[0][0]*m[2][2]*m[3][3] - m[0][0]*m[2][3]*m[3][2] - m[2][0]*m[0][2]*m[3][3] + m[2][0]*m[0][3]*m[3][2] + m[3][0]*m[0][2]*m[2][3] - m[3][0]*m[0][3]*m[2][2]
    inv[1][2] = -m[0][0]*m[1][2]*m[3][3] + m[0][0]*m[1][3]*m[3][2] + m[1][0]*m[0][2]*m[3][3] - m[1][0]*m[0][3]*m[3][2] - m[3][0]*m[0][2]*m[1][3] + m[3][0]*m[0][3]*m[1][2]
    inv[1][3] = m[0][0]*m[1][2]*m[2][3] - m[0][0]*m[1][3]*m[2][2] - m[1][0]*m[0][2]*m[2][3] + m[1][0]*m[0][3]*m[2][2] + m[2][0]*m[0][2]*m[1][3] - m[2][0]*m[0][3]*m[1][2]
    inv[2][0] = m[1][0]*m[2][1]*m[3][3] - m[1][0]*m[2][3]*m[3][1] - m[2][0]*m[1][1]*m[3][3] + m[2][0]*m[1][3]*m[3][1] + m[3][0]*m[1][1]*m[2][3] - m[3][0]*m[1][3]*m[2][1]
    inv[2][1] = -m[0][0]*m[2][1]*m[3][3] + m[0][0]*m[2][3]*m[3][1] + m[2][0]*m[0][1]*m[3][3] - m[2][0]*m[0][3]*m[3][1] - m[3][0]*m[0][1]*m[2][3] + m[3][0]*m[0][3]*m[2][1]
    inv[2][2] = m[0][0]*m[1][1]*m[3][3] - m[0][0]*m[1][3]*m[3][1] - m[1][0]*m[0][1]*m[3][3] + m[1][0]*m[0][3]*m[3][1] + m[3][0]*m[0][1]*m[1][3] - m[3][0]*m[0][3]*m[1][1]
    inv[2][3] = -m[0][0]*m[1][1]*m[2][3] + m[0][0]*m[1][3]*m[2][1] + m[1][0]*m[0][1]*m[2][3] - m[1][0]*m[0][3]*m[2][1] - m[2][0]*m[0][1]*m[1][3] + m[2][0]*m[0][3]*m[1][1]
    inv[3][0] = -m[1][0]*m[2][1]*m[3][2] + m[1][0]*m[2][2]*m[3][1] + m[2][0]*m[1][1]*m[3][2] - m[2][0]*m[1][2]*m[3][1] - m[3][0]*m[1][1]*m[2][2] + m[3][0]*m[1][2]*m[2][1]
    inv[3][1] = m[0][0]*m[2][1]*m[3][2] - m[0][0]*m[2][2]*m[3][1] - m[2][0]*m[0][1]*m[3][2] + m[2][0]*m[0][2]*m[3][1] + m[3][0]*m[0][1]*m[2][2] - m[3][0]*m[0][2]*m[2][1]
    inv[3][2] = -m[0][0]*m[1][1]*m[3][2] + m[0][0]*m[1][2]*m[3][1] + m[1][0]*m[0][1]*m[3][2] - m[1][0]*m[0][2]*m[3][1] - m[3][0]*m[0][1]*m[1][2] + m[3][0]*m[0][2]*m[1][1]
    inv[3][3] = m[0][0]*m[1][1]*m[2][2] - m[0][0]*m[1][2]*m[2][1] - m[1][0]*m[0][1]*m[2][2] + m[1][0]*m[0][2]*m[2][1] + m[2][0]*m[0][1]*m[1][2] - m[2][0]*m[0][2]*m[1][1]
    det = m[0][0]*inv[0][0] + m[0][1]*inv[1][0] + m[0][2]*inv[2][0] + m[0][3]*inv[3][0]
    if det == 0:
        return [[0]*4 for _ in range(4)]
    inv_det = 1.0 / det
    for i in range(4):
        for j in range(4):
            inv[i][j] *= inv_det
    return inv
```

**Parameters:**
- `box`: Array of 6 floats `[x_l, y_l, z_l, x_h, y_h, z_h]` [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** Dot products near zero may cause false negatives/positives due to floating-point error [G].
- **Degenerate Input:** If the projection matrix results in a zero-length normal for a plane, normalization fails [S1, unofficial].
- **Boundary Conditions:** The test returns `true` if the box touches the frustum boundary (not strictly outside) [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources. [S1, unofficial]
- **Memory Complexity:** Not given in the sources. [S1, unofficial]
- **Strengths:** Not given in the sources. [S1, unofficial]
- **Limitations:** Not given in the sources. [S1, unofficial]
- **When to use:** Not given in the sources. [S1, unofficial]

## Background
The method is adapted from Inigo Quilez's article on correct frustum culling for large bounds [S1, unofficial]. It addresses the limitation of standard frustum culling which only checks if the box is outside the frustum, failing when the box is large enough to encompass the camera or extend beyond the frustum boundaries in complex ways [S1, unofficial].

## Sources
[S1] sunjianxin/rendner_webgl, https://github.com/sunjianxin/rendner_webgl, fetched 2026-10-27 (unofficial)
