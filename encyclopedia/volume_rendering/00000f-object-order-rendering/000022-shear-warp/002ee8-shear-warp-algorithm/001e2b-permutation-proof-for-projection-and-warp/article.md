# Permutation Proof for Projection and Warp


## Overview
A mathematical proof demonstrating that the projection matrix $P$ and warp matrix $W$ can be permuted (applied as $W \cdot P \cdot S$ instead of $P \cdot W \cdot S$) in the perspective shear-warp algorithm, provided specific constraints on $W$ are met [S1, unofficial]. This property is a basic premise for the perspective projection shear-warp algorithm [S1, unofficial].

## How it works
### Inputs and Outputs
- **Inputs**: Eye position in object space $e_o$, overall viewing matrix $M_{ov}$, projection matrix $P$, warp matrix $W$, shear matrix $S$ [S1, unofficial].
- **Outputs**: Verification that $w_{02} = 0$ and $w_{12} = 0$ in the warp matrix $W$ when derived from $M_{ov}$ and $e_o$, proving the permutation holds [S1, unofficial].

### Data Structures
The method operates on $4 \times 4$ homogeneous transformation matrices [S1, unofficial].
- **Projection Matrix $P$**:
  $$
  P = \begin{pmatrix}
  1 & 0 & 0 & 0 \\
  0 & 1 & 0 & 0 \\
  0 & 0 & 0 & 0 \\
  0 & 0 & 0 & 1
  \end{pmatrix}
  $$
  [S1, unofficial]
- **Shear Matrix $S$**:
  $$
  S = \begin{pmatrix}
  1 & 0 & e_{x,z} & 0 \\
  0 & 1 & e_{y,z} & 0 \\
  0 & 0 & 1 & 0 \\
  0 & 0 & e_{w,z} & 1
  \end{pmatrix}
  $$
  [S1, unofficial]
  Where:
  $$
  e_{x,z} = -\frac{e_o^x}{e_o^z}, \quad e_{y,z} = -\frac{e_o^y}{e_o^z}, \quad e_{w,z} = -\frac{e_o^w}{e_o^z}
  $$
  [S1, unofficial]
- **Warp Matrix $W$**:
  $$
  W = \begin{pmatrix}
  w_{00} & w_{01} & w_{02} & w_{03} \\
  w_{10} & w_{11} & w_{12} & w_{13} \\
  w_{20} & w_{21} & w_{22} & w_{23} \\
  w_{30} & w_{31} & w_{32} & w_{33}
  \end{pmatrix}
  $$
  [S1, unofficial]

### Steps
1. Compute object space eye position $e_o = M_{wo} \cdot (0, 0, -1, 0)^T$ [S1, unofficial].
2. Define $P$, $S$, and $W$ as specified in the Data Structures section [S1, unofficial].
3. The standard viewing matrix is defined as $M_{ov} = P \cdot W \cdot S$ [S1, unofficial].
4. Compute the product $V = P \cdot W \cdot S$. The elements at indices $(0,2)$ and $(1,2)$ are $V_{02} = w_{00}e_{x,z} + w_{01}e_{y,z} + w_{03}e_{w,z}$ and $V_{12} = w_{10}e_{x,z} + w_{11}e_{y,z} + w_{13}e_{w,z}$ [S1, unofficial].
5. Compute $M_{ov}$ elements at $(0,2)$ and $(1,2)$: $M_{ov}^{02} = w_{00}e_{x,z} + w_{01}e_{y,z} + w_{02} + w_{03}e_{w,z}$ and $M_{ov}^{12} = w_{10}e_{x,z} + w_{11}e_{y,z} + w_{12} + w_{13}e_{w,z}$ [S1, unofficial].
6. For identical results, $w_{02} = 0$ and $w_{12} = 0$ [S1, unofficial].
7. Express $w_{02}$ and $w_{12}$ using $M_{ov}$: $w_{02} = M_{ov}^{00}e_{x,z} + M_{ov}^{01}e_{y,z} + M_{ov}^{02} + M_{ov}^{03}e_{w,z}$ and $w_{12} = M_{ov}^{10}e_{x,z} + M_{ov}^{11}e_{y,z} + M_{ov}^{12} + M_{ov}^{13}e_{w,z}$ [S1, unofficial].
8. Multiply by $e_o^z$: $w_{02}e_o^z = M_{ov}^{00}e_o^x + M_{ov}^{01}e_o^y + M_{ov}^{02}e_o^z + M_{ov}^{03}e_o^w$ [S1, unofficial]. This equals zero because $M_{ow} \cdot (0,0,-1,0)^T = e_o$ implies the transformed eye position has zero x and y components relative to the camera center [S1, unofficial]. Thus $w_{02} = 0$ and $w_{12} = 0$ [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def verify_permutation(M_ov, e_o):
    """
    Verifies that w02 and w12 are zero for the warp matrix derived from M_ov and e_o.
    
    Args:
        M_ov (list of lists): 4x4 overall viewing matrix (object to view space).
        e_o (list): 4-vector eye position in object space [ex, ey, ez, ew].
        
    Returns:
        bool: True if w02 and w12 are zero.
    """
    ex, ey, ez, ew = e_o
    if ez == 0:
        raise ValueError("Division by zero: eye z-component is zero.")
    ex_z = -ex / ez
    ey_z = -ey / ez
    ew_z = -ew / ez
    
    w02 = M_ov[0][0]*ex_z + M_ov[0][1]*ey_z + M_ov[0][2] + M_ov[0][3]*ew_z
    w12 = M_ov[1][0]*ex_z + M_ov[1][1]*ey_z + M_ov[1][2] + M_ov[1][3]*ew_z
    
    return w02 == 0.0 and w12 == 0.0
```

### Edge Cases and Pitfalls
- **$e_o^z = 0$**: If the eye position's z-component in object space is zero, the shear parameters $e_{x,z}$, $e_{y,z}$, and $e_{w,z}$ are undefined due to division by zero [G].
- **Exact Arithmetic**: The proof relies on exact zero values [G]. In implementation, floating point errors may result in $w_{02}$ and $w_{12}$ being very small but non-zero, requiring a tolerance check [G].

## Operating characteristics
Not given in the sources.

## Background
The perspective shear-warp algorithm decomposes the viewing transformation into shear, projection, and warp steps [S1, unofficial]. The permutation of the projection and warp matrices was a basic premise of the algorithm but lacked a formal proof until the work described in [S1, unofficial].

## Sources
[S1] The Perspective Shear-Warp Algorithm in a Virtual Environment., https://www.researchgate.net/publication/220943822_The_Perspective_Shear-Warp_Algorithm_in_a_Virtual_Environment, fetched 2026-10-29 (unofficial)
