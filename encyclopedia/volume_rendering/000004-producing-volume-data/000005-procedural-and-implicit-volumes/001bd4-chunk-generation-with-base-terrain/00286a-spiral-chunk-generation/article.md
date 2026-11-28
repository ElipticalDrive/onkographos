# Spiral Chunk Generation

Also called: Procedural Spiral Placement, SpiralGenerate

## Overview
Spiral Chunk Generation produces a queue of 3D chunk positions in a spiral pattern or vertical columns to drive procedural octree and mesh generation [S1, unofficial]. It differs from sibling methods by explicitly decoupling the spatial traversal order from the chunk processing pipeline via a thread-safe queue [S1, unofficial].

## How it works
**Inputs:**
- `width`, `height`: Dimensions of the generation area in grid units [S1, unofficial].
- `m_chunkSize`: Size of each chunk in world units [S1, unofficial].
- `generateColumns`: Boolean flag toggling between column-first and layer-first generation order [S1, unofficial].

**Outputs:**
- A populated `chunkQueue` containing `glm::ivec3` positions [S1, unofficial].

**Data Structures:**
- `chunkQueue`: A `std::queue<glm::ivec3>` storing pending chunk positions, protected by `queueMutex` [S1, unofficial].

**Algorithm Steps:**
1. Initialize grid size: `gridSize = 2 * width + 1` [S1, unofficial].
2. Define 2D movement directions in the XZ plane: `right = (0, 0, m_chunkSize)`, `down = (m_chunkSize, 0, 0)`, `left = (0, 0, -m_chunkSize)`, `up = (-m_chunkSize, 0, 0)` [S1, unofficial].
3. Initialize state: `basePosition = (0, 0, 0)`, `directionIndex = 0`, `step = 1`, `count = 0` [S1, unofficial].
4. If `generateColumns` is true: loop `i` from 1 to `gridSize * gridSize` [S1, unofficial]. Inner loop `y` from `height / 2` down to `-height / 2` [S1, unofficial]. Queue position `basePosition + (0, y * m_chunkSize, 0)` [S1, unofficial]. If `count == step`, increment `directionIndex`; if `directionIndex` is even, increment `step`; reset `count = 0` [S1, unofficial]. Add `directions[directionIndex]` to `basePosition` and increment `count` [S1, unofficial].
5. If `generateColumns` is false: loop `y` from `-height / 2` to `height / 2` [S1, unofficial]. Set `levelBasePosition = (0, y * m_chunkSize, 0)` and reset `basePosition`, `directionIndex`, `step`, `count` [S1, unofficial]. Inner loop `i` from 1 to `gridSize * gridSize` [S1, unofficial]. Queue `basePosition` [S1, unofficial]. Update spiral state identically to the column mode [S1, unofficial].
6. Start a detached thread to process the queue via `ProcessQueue` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import threading
from collections import deque
from typing import Tuple, List

def spiral_chunk_generation(width: int, height: int, m_chunkSize: int, generate_columns: bool) -> deque:
    chunk_queue = deque()
    grid_size = 2 * width + 1
    directions: List[Tuple[int, int, int]] = [
        (0, 0, m_chunkSize),   # right
        (m_chunkSize, 0, 0),   # down
        (0, 0, -m_chunkSize),  # left
        (-m_chunkSize, 0, 0)   # up
    ]
    base_position: Tuple[int, int, int] = (0, 0, 0)
    direction_index: int = 0
    step: int = 1
    count: int = 0

    def update_state() -> None:
        nonlocal direction_index, step, count, base_position
        if count == step:
            direction_index += 1
            if direction_index % 2 == 0:
                step += 1
            count = 0
        dx, dy, dz = directions[direction_index % 4]
        base_position = (base_position[0] + dx, base_position[1] + dy, base_position[2] + dz)
        count += 1

    if generate_columns:
        for _ in range(1, grid_size * grid_size + 1):
            for y in range(height // 2, -height // 2 - 1, -1):
                chunk_queue.append((base_position[0], base_position[1] + y * m_chunkSize, base_position[2]))
            update_state()
    else:
        for y in range(-height // 2, height // 2 + 1):
            level_base_position = (0, y * m_chunkSize, 0)
            base_position = (0, 0, 0)
            direction_index = 0
            step = 1
            count = 0
            for _ in range(1, grid_size * grid_size + 1):
                chunk_queue.append(base_position)
                update_state()
    # Detached thread started for ProcessQueue [S1, unofficial]
    return chunk_queue
```

**Parameters:**
- `width`, `height`: Dimensions of the generation area in grid units [S1, unofficial].
- `generateColumns`: Boolean flag to toggle between column-first and layer-first generation order [S1, unofficial].
- `m_chunkSize`: Size of each chunk in world units [S1, unofficial].

**Edge Cases and Pitfalls:**
- The `EnlargeCorners` function computes an offset vector that is explicitly marked as unused in comments and returned in a tuple but never consumed in the visible control flow [S1, unofficial].

## Operating characteristics
Not given in the sources.

## Background
The method originates from the `Tsarpf/procgen` repository [S1, unofficial].

## Sources
[S1] Tsarpf/procgen, https://github.com/Tsarpf/procgen, fetched 2026-10-26 (unofficial)
