# Fast Voxel Traversal 2D

Also called: 2D grid traversal, 2D voxel traversal

## Overview
Fast Voxel Traversal 2D is an algorithm to traverse all 2D grid cells (voxels) that a line segment passes through, used for ray tracing acceleration and line-voxel intersection [S1, unofficial]. It differs from Bresenham's line algorithm, which produces an aesthetic approximation of a line but intentionally omits cells the line passes through to avoid visual 'ugliness'; Fast Voxel Traversal aims to visit all cells intersected by the line [S1, unofficial].

## How it works
**Inputs:** Start point $(sX, sY)$, end point $(eX, eY)$, grid origin $(x0, y0)$, cell dimensions $(dx, dy)$.
**Outputs:** List of voxel indices $(x, y)$ visited by the segment.

**Data Structures:**
- `geom`: Array of parameters. `geom[0]` = $x0$, `geom[1]` = $dx$, `geom[3]` = $y0$, `geom[5]` = $dy$.
- `ptsIndexes`: List of integer coordinate pairs.

**Parameters:**
- `Max_Delta`: Maximum allowed value for $tDelta$ to prevent division by zero or overflow. Default: $1000000.0$ [S1, unofficial].

**Steps:**
1.  **Initialize Indices:**
    Compute start indices:
    $$sXIndex = \frac{sX - x0}{dx}$$
    $$sYIndex = \frac{sY - y0}{dy}$$
    Compute end indices:
    $$eXIndex = \frac{eX - sXIndex}{dx} + sXIndex$$
    $$eYIndex = \frac{eY - sYIndex}{dy} + sYIndex$$
    [S1, unofficial]

2.  **Compute Deltas and Steps:**
    $$deltaX = eXIndex - sXIndex$$
    $$stepX = \begin{cases} 1 & \text{if } deltaX > 0 \\ -1 & \text{if } deltaX < 0 \\ 0 & \text{otherwise} \end{cases}$$
    $$tDeltaX = \begin{cases} \min\left(\frac{stepX}{deltaX}, Max\_Delta\right) & \text{if } stepX \neq 0 \\ Max\_Delta & \text{otherwise} \end{cases}$$
    [S1, unofficial]

    $$deltaY = eYIndex - sYIndex$$
    $$stepY = \begin{cases} 1 & \text{if } deltaY > 0 \\ -1 & \text{if } deltaY < 0 \\ 0 & \text{otherwise} \end{cases}$$
    $$tDeltaY = \begin{cases} \min\left(\frac{stepY}{deltaY}, Max\_Delta\right) & \text{if } stepY \neq 0 \\ Max\_Delta & \text{otherwise} \end{cases}$$
    [S1, unofficial]

3.  **Compute Initial Distances to Boundaries:**
    Voxels span from $0.0$ to $1.0$ (grid-aligned), not $-0.5$ to $0.5$ (center-aligned) [S1, unofficial].
    $$maxX = \begin{cases} tDeltaX \cdot (1 - sXIndex + \lfloor sXIndex \rfloor) & \text{if } stepX > 0 \\ tDeltaX \cdot (sXIndex - \lfloor sXIndex \rfloor) & \text{otherwise} \end{cases}$$
    $$maxY = \begin{cases} tDeltaY \cdot (1 - sYIndex + \lfloor sYIndex \rfloor) & \text{if } stepY > 0 \\ tDeltaY \cdot (sYIndex - \lfloor sYIndex \rfloor) & \text{otherwise} \end{cases}$$
    [S1, unofficial]

4.  **Traverse:**
    Initialize $x = sXIndex$, $y = sYIndex$.
    Add $\text{round}(x), \text{round}(y)$ to output. Let $prevPt = [\text{round}(x), \text{round}(y)]$.
    Loop:
    - If $maxX < maxY$:
      $$maxX += tDeltaX$$
      $$x += stepX$$
    - Else:
      $$maxY += tDeltaY$$
      $$y += stepY$$
    - Compute $pt = [\text{round}(x), \text{round}(y)]$.
    - If $pt \neq prevPt$:
      Append $pt$ to output.
      $prevPt = pt$.
    - Break if $maxX > 1$ and $maxY > 1$.
    [S1, unofficial]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fast_voxel_traversal_2d(strPt, endPt, geom, Max_Delta=1000000.0):
    x0 = geom[0]
    dx = geom[1]
    y0 = geom[3]
    dy = geom[5]

    sX, sY = strPt
    eX, eY = endPt

    sXIndex = (sX - x0) / dx
    sYIndex = (sY - y0) / dy
    
    # Note: Formula from source [S1, unofficial]
    eXIndex = ((eX - sXIndex) / dx) + sXIndex
    eYIndex = ((eY - sYIndex) / dy) + sYIndex

    deltaX = float(eXIndex - sXIndex)
    deltaXSign = 1 if deltaX > 0 else (-1 if deltaX < 0 else 0)
    stepX = deltaXSign

    if deltaXSign != 0:
        tDeltaX = min(deltaXSign / deltaX, Max_Delta)
    else:
        tDeltaX = Max_Delta

    if deltaXSign > 0:
        maxX = tDeltaX * (1 - sXIndex + int(sXIndex))
    else:
        maxX = tDeltaX * (sXIndex - int(sXIndex))

    deltaY = float(eYIndex - sYIndex)
    deltaYSign = 1 if deltaY > 0 else (-1 if deltaY < 0 else 0)
    stepY = deltaYSign

    if deltaYSign != 0:
        tDeltaY = min(deltaYSign / deltaY, Max_Delta)
    else:
        tDeltaY = Max_Delta

    if deltaYSign > 0:
        maxY = tDeltaY * (1 - sYIndex + int(sYIndex))
    else:
        maxY = tDeltaY * (sYIndex - int(sYIndex))

    x = sXIndex
    y = sYIndex

    ptsIndexes = []
    pt = [round(x), round(y)]
    ptsIndexes.append(pt)
    prevPt = pt

    while True:
        if maxX < maxY:
            maxX += tDeltaX
            x += deltaXSign
        else:
            maxY += tDeltaY
            y += deltaYSign

        pt = [round(x), round(y)]
        if pt != prevPt:
            ptsIndexes.append(pt)
            prevPt = pt

        if maxX > 1 and maxY > 1:
            break

    return ptsIndexes
```

**Edge Cases and Pitfalls:**
- **Voxel Coordinate System Mismatch:** Voxels span from $0.0$ to $1.0$ (grid-aligned), not $-0.5$ to $0.5$ (center-aligned). Assuming center-aligned voxels causes incorrect initial $maxX$ and $maxY$ calculations [S1, unofficial].
- **Diagonal Step Cell Omission:** When the traversal makes a diagonal step, it may skip intermediate cells that the line actually passes through. This requires additional line-rectangle intersection checks to recover missed cells [S1, unofficial].

## Operating characteristics
Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] Fast Voxel Traversal 2D, https://stackoverflow.com/questions/41290526/fast-voxel-traversal-2d, fetched 2026-10-24 (unofficial)
