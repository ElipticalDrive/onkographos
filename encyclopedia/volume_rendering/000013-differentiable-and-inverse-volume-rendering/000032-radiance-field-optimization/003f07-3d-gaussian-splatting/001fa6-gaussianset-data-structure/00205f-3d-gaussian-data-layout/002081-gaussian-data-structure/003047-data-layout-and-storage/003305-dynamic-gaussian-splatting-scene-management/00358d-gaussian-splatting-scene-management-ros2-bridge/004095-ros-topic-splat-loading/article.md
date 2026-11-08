# ROS Topic Splat Loading

Also called: RosTopicSource

## Overview
A method to load 3D Gaussian splats from a ROS topic by subscribing to a `SplatArray` message, converting the wire format into a GPU-ready `SplatGPU` struct layout [S1, unofficial]. It differs from static loading by enabling real-time scene updates via ROS2 communication, bridging simulation or sensor data directly into the Gaussian Splatting renderer [S1, unofficial].

## How it works
**Inputs:** A ROS2 `SplatArray` message containing a list of splats, each with pose, opacity, covariance, and spherical harmonics.
**Outputs:** A `LoadResult` structure containing a vector of `SplatGPU` structs and the scene's SH degree.

**Data Structures:**
*   `SplatGPU`: The internal GPU-ready representation.
    *   `center[3]`: `float` array for position $(x, y, z)$.
    *   `alpha`: `float` for opacity, normalized to $[0, 1]$.
    *   `covA[3]`: `float` array storing the first row of the symmetric $3 \times 3$ covariance matrix.
    *   `covB[3]`: `float` array storing the second row of the symmetric $3 \times 3$ covariance matrix.
    *   `sh[num_coefs][4]`: `float` array for spherical harmonics coefficients. `sh[k][0..2]` stores RGB for degree $k$, `sh[k][3]` is padding (0.0).
*   `LoadResult`:
    *   `splats`: `std::vector<SplatGPU>`
    *   `sh_degree`: `uint32_t`

**Wire Format Mapping:**
The input `SplatArray` message contains individual splat messages. The mapping from ROS message fields to `SplatGPU` fields is as follows [S1, unofficial]:
*   `center`: Extracted from `s.pose.pose.pose.position.x/y/z`.
*   `alpha`: Extracted from `s.opacity` (uint8, range 0–255) via $alpha = opacity / 255.0$.
*   `covA`: Extracted from `s.pose.pose.covariance` (6x6 row-major array) at indices 0, 1, 2.
*   `covB`: Extracted from `s.pose.pose.covariance` at indices 7, 8, 14. These indices correspond to the top-left $3 \times 3$ symmetric block of the covariance matrix.
*   `sh`: Extracted from `s.spherical_harmonics` (coef-major, RGB-interleaved). For coefficient $k$, `sh[k][0..2]` maps to `spherical_harmonics[3*k+0..2]`.

**Algorithm:**
1.  Initialize `LoadResult out`.
2.  Determine `sh_degree`:
    *   If `msg.splats` is not empty, set `sh_degree = msg.splats[0].harmonics_deg`.
    *   Else, set `sh_degree = 0`.
3.  Calculate `num_coefs = (sh_degree + 1)^2`.
4.  Reserve memory for `out.splats` based on `msg.splats.size()`.
5.  For each splat `s` in `msg.splats`:
    a.  Create `SplatGPU g`.
    b.  Set `g.center[0] = (float)s.pose.pose.pose.position.x`.
    c.  Set `g.center[1] = (float)s.pose.pose.pose.position.y`.
    d.  Set `g.center[2] = (float)s.pose.pose.pose.position.z`.
    e.  Set `g.alpha = (float)s.opacity / 255.0f`.
    f.  Extract covariance from `s.pose.pose.covariance` array (let this be `cov`):
        i.  `g.covA[0] = (float)cov[0]`
        ii. `g.covA[1] = (float)cov[1]`
        iii. `g.covA[2] = (float)cov[2]`
        iv. `g.covB[0] = (float)cov[7]`
        v.  `g.covB[1] = (float)cov[8]`
        vi. `g.covB[2] = (float)cov[14]`
    g.  Loop `k` from 0 to `num_coefs - 1`:
        i.  `g.sh[k][0] = (float)s.spherical_harmonics[3*k + 0]`
        ii. `g.sh[k][1] = (float)s.spherical_harmonics[3*k + 1]`
        iii. `g.sh[k][2] = (float)s.spherical_harmonics[3*k + 2]`
        iv. `g.sh[k][3] = 0.0f`
    h.  Push `g` to `out.splats`.
6.  Set `out.sh_degree = sh_degree`.
7.  Return `out`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def load_ros_splats(msg):
    # 1. Initialize result
    out = LoadResult()
    
    # 2. Determine SH degree
    if len(msg.splats) > 0:
        sh_degree = msg.splats[0].harmonics_deg
    else:
        sh_degree = 0
        
    # 3. Calculate number of coefficients
    num_coefs = (sh_degree + 1) ** 2
    
    # 4. Reserve space
    out.splats.reserve(len(msg.splats))
    
    # 5. Process each splat
    for s in msg.splats:
        g = SplatGPU()
        
        # Position
        g.center[0] = float(s.pose.pose.pose.position.x)
        g.center[1] = float(s.pose.pose.pose.position.y)
        g.center[2] = float(s.pose.pose.pose.position.z)
        
        # Opacity
        g.alpha = float(s.opacity) / 255.0
        
        # Covariance (from 6x6 row-major array)
        cov = s.pose.pose.covariance
        g.covA[0] = float(cov[0])
        g.covA[1] = float(cov[1])
        g.covA[2] = float(cov[2])
        g.covB[0] = float(cov[7])
        g.covB[1] = float(cov[8])
        g.covB[2] = float(cov[14])
        
        # Spherical Harmonics
        sh_data = s.spherical_harmonics
        for k in range(num_coefs):
            base_idx = 3 * k
            g.sh[k][0] = float(sh_data[base_idx + 0])
            g.sh[k][1] = float(sh_data[base_idx + 1])
            g.sh[k][2] = float(sh_data[base_idx + 2])
            g.sh[k][3] = 0.0
            
        out.splats.append(g)
        
    out.sh_degree = sh_degree
    return out
```

**Parameters:**
*   QoS Profile: `reliable()` transport and `transient_local()` durability [S1, unofficial].

**Edge Cases and Pitfalls:**
*   Assumes uniform `harmonics_deg` across all splats in the message [S1, unofficial].
*   Assumes `deg <= 3` [S1, unofficial].
*   If `spherical_harmonics` length does not match $3 \times (deg+1)^2$, behavior is undefined [S1, unofficial].

## Operating characteristics
Not given in the sources.

## Background
This method is part of the `gsplat_rviz_plugin` within the `splat-ros` ecosystem, designed to integrate 3D Gaussian Splatting visualization into ROS2-based robotics frameworks [S1, unofficial]. It enables real-time rendering of dynamic Gaussian scenes published over ROS topics.

## Sources
[S1] splat-ros/RVizSplat, https://github.com/splat-ros/RVizSplat, fetched 2026-10-27 (unofficial)
