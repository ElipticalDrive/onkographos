# Volume Visualization Camera

Also called: View Matrix Generator, Volume Camera

## Overview
A camera controller for volume visualization that computes a view matrix by applying volume scaling, centering, rotation, zoom, and translation transformations [S1, unofficial]. It differs from standard scene cameras by explicitly modeling the volume's physical bounds and scale, ensuring the view matrix correctly maps volume coordinates to clip space regardless of the dataset's dimensions [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Volume bounds (size), volume scale, camera position, zoom factors, rotation deltas, and pan deltas.
*   **Outputs:** A 4x4 view matrix.

**Data Structures:**
*   `_forward`: `vec3`, initialized to `[0, 0, 1]` [S1, unofficial].
*   `_up`: `vec3`, initialized to `[0, 1, 0]` [S1, unofficial].
*   `_zoom`: `vec3`, initialized to `[1, 1, 1]` [S1, unofficial].
*   `_position`: `vec3`, initialized to `[0, 0, -length(bounds)]` [S1, unofficial].
*   `volumeCentre`: `vec3`, computed as `0.5 * volume.bounds` [S1, unofficial].

**View Matrix Construction:**
The view matrix is constructed by multiplying transformations in the following order [S1, unofficial]:
1.  Initialize `view` to identity.
2.  Multiply by translation of `_position`.
3.  Multiply by scaling of `_zoom`.
4.  Multiply by rotation matrix.
5.  Multiply by translation of `volumeCentre`.
6.  Multiply by scaling of `volumeScale`.

**Rotation Matrix Computation:**
1.  Compute `right` vector via cross product of `_up` and `_forward`.
2.  Normalize `right` vector.
3.  Construct 4x4 matrix with columns `[right, _up, _forward, 0]`.
4.  Transpose the matrix [S1, unofficial].

**Camera Rotation Update:**
1.  Compute `right` vector via cross product of `_forward` and `_up`.
2.  Create rotation matrix `rotY` around `_up` by `delta[0]`.
3.  Transform `_forward` by `rotY`.
4.  Create rotation matrix `rotX` around `right` by `delta[1]`.
5.  Transform `_up` and `_forward` by `rotX` [S1, unofficial].

**Light Direction Update:**
1.  Create rotation matrix `rotX` around `[0,1,0]` by `delta[0]`.
2.  Create rotation matrix `rotY` around `[1,0,0]` by `delta[1]`.
3.  Transform `_lightDir` by `rotX` then `rotY` [S1, unofficial].

**Zoom Clamping:**
If `_zoom[0] - delta > 0`, subtract `delta` from `_zoom[0]` and `_zoom[1]` [S1, unofficial].

**Pan Update:**
Subtract `delta[0]` from `_position[0]` and `delta[1]` from `_position[1]` [S1, unofficial].

**Recentre:**
Set `_position[0]` to `-size[0]/2` and `_position[1]` to `-size[1]/2` [S1, unofficial].

## Operating characteristics
Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] fraserlove/ossium, https://github.com/fraserlove/ossium, fetched 2026-10-27 (unofficial)
