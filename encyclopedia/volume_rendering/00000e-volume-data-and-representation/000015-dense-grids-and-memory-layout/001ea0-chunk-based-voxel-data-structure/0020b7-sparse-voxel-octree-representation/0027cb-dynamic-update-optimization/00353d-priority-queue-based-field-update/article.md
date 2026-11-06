# Priority Queue Based Field Update

Also called: Level Manager, Coarse-to-Fine Update Queue

## Overview
Priority Queue Based Field Update is a dynamic optimization method for sparse voxel octrees that ensures volume field updates (such as sculpting or erosion) proceed from coarser to finer levels. This ordering is essential for interactive display, as it prevents visual artifacts caused by updating fine details before their coarser parents are consistent. It differs from recursive depth-first approaches, which miss this coarse-to-fine requirement, by using a priority queue sorted by cell level and tool identity to manage the update sequence [S1].

## How it works
### Inputs and Outputs
- **Input**: A Tool/Action couple (e.g., "Erode", "Grow") applied to the volume.
- **Output**: A modified Sparse Voxel Octree where cells intersected by the tool are updated and subdivided/simplified as needed, maintaining the coarse-to-fine consistency.

### Data Structures
The method relies on three primary data structures, enclosed within a **Level Manager**:

1.  **ToolCopy**: A frozen snapshot of the tool at the moment of application.
    -   **Fields**:
        -   State information: Position, orientation, scale, color, etc.
        -   Reference: Pointer to the original tool and the specific action involved.
        -   Unique Identifier: An integer acting as a time-counter/timestamp to avoid redundant computations [S1].
2.  **Cell-Queue**: A simple FIFO queue used to temporarily store cells intersected by the tool at a specific level.
    -   **Operations**: Constant-time `pushback` and `popfront` [S1].
    -   **Indexing**: Indexed by the `ToolCopy` and the size (level) of the cells it contains.
    -   **Access**: Only the update thread accesses the Cell-queues it empties or fills. The draw process accesses cells directly through the hierarchy, so no locking is needed inside Cell-queues [S1].
3.  **Level Manager Priority Queue**: The central scheduler.
    -   **Elements**: Cell-queues (not individual ToolCopies).
    -   **Sorting Criteria**: Sorted by **Level** (cell size, coarser first) and then by **Tool/Action** (sequential application) [S1].
    -   **Implementation**: Typically implemented using a standard library priority queue (e.g., STL `priority_queue` relying on `vector`) [S1].

### Algorithm Steps

1.  **Initialization (User Thread)**:
    -   When a user applies a tool, the system creates a `ToolCopy` from the Tool/Action couple.
    -   A new `Cell-Queue` is created, containing only the current root cell.
    -   This `Cell-Queue` is inserted into the `Level Manager` priority queue [S1].

2.  **Update Loop (Update Thread)**:
    -   The update thread runs continuously in the background.
    -   It extracts the highest-priority `Cell-Queue` from the `Level Manager`.
    -   It processes cells in the queue using the modified `Apply` procedure (see below).
    -   If the priority queue becomes empty, the thread performs a **Simplification** step (as defined in the parent method) and then suspends itself using a blocking lock mechanism to save processor resources [S1].

3.  **Modified Apply Procedure**:
    -   The `Apply` procedure receives a `Cell-Queue` as an extra parameter.
    -   It updates the internal vertices of the current cell and subdivides it if necessary (based on the tool's effect).
    -   **Crucial Difference**: Instead of recursively calling `Apply` on existing children immediately, it identifies children that intersect the tool.
    -   These intersecting children are appended to the `Cell-Queue` associated with the next finer level.
    -   If the `Cell-Queue` for the next level does not exist in the manager, it is created and inserted into the `Level Manager` priority queue [S1].

4.  **Interruption Handling**:
    -   If a coarser level is inserted into the `Level Manager` (e.g., from a new user action or undo/redo) while the update thread is processing a finer level's `Cell-Queue`, the current processing is interrupted.
    -   The interrupted `Cell-Queue` is re-inserted into the `Level Manager` priority queue.
    -   When the update thread returns to this queue, it resumes handling from where it was suspended [S1].

5.  **Threading Synchronization**:
    -   Two threads are used: one for user interaction/display and one for background updates [S1].
    -   Blocking access (locking) is limited strictly to the insertion and removal of `Cell-Queues` in the `Level Manager` priority queue.
    -   No exclusion mechanism is needed inside `Cell-Queues` or for accessing cells through the hierarchy, as the update process is the only one modifying the queue structure, and the draw process reads cells directly [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import threading
import queue
import heapq

# Constants
LEVEL_COARSE = 0
LEVEL_FINE = MAX_DEPTH

class ToolCopy:
    """Frozen copy of a tool at application time."""
    def __init__(self, tool_ref, action, timestamp, state):
        self.tool_ref = tool_ref
        self.action = action
        self.timestamp = timestamp
        self.state = state  # position, orientation, scale, color, etc.

class CellQueue:
    """Queue of cells at a specific level for a specific ToolCopy."""
    def __init__(self, tool_copy, level):
        self.tool_copy = tool_copy
        self.level = level
        self.cells = []  # List of cell references

    def push(self, cell):
        self.cells.append(cell)

    def pop(self):
        return self.cells.pop(0)

    def is_empty(self):
        return len(self.cells) == 0

class LevelManager:
    """Manages the priority queue of Cell-Queues."""
    def __init__(self):
        # Priority queue stores tuples: (level, tool_timestamp, cell_queue)
        # Sorted by level (ascending), then by timestamp (ascending)
        self._pq = []
        self._lock = threading.Lock()
        self._condition = threading.Condition(self._lock)
        self._running = False
        self._update_thread = None

    def start(self):
        self._running = True
        self._update_thread = threading.Thread(target=self._update_loop)
        self._update_thread.start()

    def stop(self):
        self._running = False
        with self._condition:
            self._condition.notify_all()
        if self._update_thread:
            self._update_thread.join()

    def insert_tool(self, tool_copy, root_cell):
        """Called by user thread when a tool is applied."""
        level = root_cell.level
        cell_queue = CellQueue(tool_copy, level)
        cell_queue.push(root_cell)
        
        with self._condition:
            # Push to priority queue: (level, timestamp, cell_queue)
            heapq.heappush(self._pq, (level, tool_copy.timestamp, cell_queue))
            self._condition.notify()

    def _update_loop(self):
        while self._running:
            cell_queue = None
            with self._condition:
                # Wait until there is work or we are stopping
                while not self._pq and self._running:
                    self._condition.wait()
                
                if not self._running:
                    break
                
                if self._pq:
                    # Pop highest priority (coarsest level, earliest timestamp)
                    level, ts, cell_queue = heapq.heappop(self._pq)
            
            if cell_queue:
                self._process_cell_queue(cell_queue)
            
            # If queue is empty, perform simplification and suspend
            with self._condition:
                if not self._pq:
                    # Perform simplification step (defined in parent method)
                    perform_simplification()
                    # Suspend to save resources
                    self._condition.wait()

    def _process_cell_queue(self, cell_queue):
        """Process all cells in the queue, inserting children into next level queues."""
        while not cell_queue.is_empty():
            cell = cell_queue.pop()
            self._apply_tool_to_cell(cell, cell_queue)

    def _apply_tool_to_cell(self, cell, current_queue):
        """
        Modified Apply procedure.
        Updates cell and subdivides if needed.
        Inserts intersecting children into the appropriate next-level Cell-Queue.
        """
        # 1. Update internal vertices and check subdivision criteria
        needs_subdivision = check_subdivision(cell)
        apply_tool_effect(cell)

        if needs_subdivision:
            # 2. Identify children that intersect the tool
            children = get_children(cell)
            intersecting_children = []
            for child in children:
                if intersects_tool(child, current_queue.tool_copy):
                    intersecting_children.append(child)

            # 3. Insert intersecting children into the next level's Cell-Queue
            next_level = cell.level + 1
            tool_copy = current_queue.tool_copy
            
            # Find or create the Cell-Queue for the next level
            next_queue = find_or_create_queue(tool_copy, next_level)
            
            # Insert into priority queue if it's a new queue
            if next_queue.was_just_created:
                with self._condition:
                    heapq.heappush(self._pq, (next_level, tool_copy.timestamp, next_queue))
                    self._condition.notify()
            
            # Add children to the queue
            for child in intersecting_children:
                next_queue.push(child)

def find_or_create_queue(tool_copy, level):
    """Helper to find existing queue or create new one."""
    # Implementation detail: In a real system, this might scan the PQ or use a hash map.
    # For simplicity, we assume the Level Manager maintains a lookup or we scan.
    # Note: The source implies direct insertion/handling inside the manager.
    # We will assume a simple scan for the purpose of this pseudocode, 
    # though a hash map would be more efficient.
    # However, the source says "Cell-queues are indexed by a ToolCopy and the size".
    # So we can look up by (tool_copy, level).
    
    # [G] Assuming a dictionary lookup for efficiency, as scanning PQ is O(N).
    # The source doesn't specify the lookup mechanism, only that they are indexed.
    if (tool_copy, level) in global_queue_registry:
        return global_queue_registry[(tool_copy, level)]
    else:
        new_queue = CellQueue(tool_copy, level)
        new_queue.was_just_created = True
        global_queue_registry[(tool_copy, level)] = new_queue
        return new_queue

# Global registry for queue lookup [G]
global_queue_registry = {}

def perform_simplification():
    """Placeholder for simplification logic defined in parent method."""
    pass

def check_subdivision(cell):
    # Returns True if cell needs to be split
    pass

def apply_tool_effect(cell):
    # Modifies cell data
    pass

def get_children(cell):
    # Returns list of child cells
    pass

def intersects_tool(child, tool_copy):
    # Checks if child geometry intersects the tool's frozen state
    pass
```

### Parameters and Edge Cases
-   **Timestamps**: The unique identifier in `ToolCopy` acts as a time-counter. This ensures that if the same tool is applied multiple times, the order is preserved, and redundant computations can be avoided [S1].
-   **Interrupts**: If a coarser level is inserted while processing a finer level, the finer level's queue is suspended and re-queued. This ensures coarse-to-fine order is strictly maintained even with interleaved user actions [S1].
-   **Simplification**: When the queue is empty, a simplification step is performed. This step is not detailed in this article but is part of the parent method [S1].
-   **Thread Safety**: Locking is only required for the `Level Manager` priority queue operations. No locks are needed for cell data access or `Cell-Queue` internal operations [S1].

## Operating characteristics
-   **Time Complexity**: The priority queue operations (insertion and removal) are $O(\log N)$, where $N$ is the number of cell-queues. The `Apply` procedure processes each cell once per level. The total time is proportional to the number of cells intersected by the tool across all levels.
-   **Memory Complexity**: Additional memory is required for `ToolCopy` objects and `Cell-Queue` structures. The `Cell-Queue` structures are temporary and are consumed as the update proceeds.
-   **Performance**: The two-thread approach (update thread + user thread) minimizes blocking. The update thread suspends when idle, saving CPU resources. The locking overhead is minimal because access to the `Level Manager` is infrequent compared to cell processing [S1].
-   **Strengths**: Ensures correct coarse-to-fine update order for interactive display. Handles interruptions gracefully. Simple synchronization model.
-   **Limitations**: Requires a priority queue data structure. The recursive approach is simpler but incorrect for interactive display. The method assumes a single update thread; handling multiple concurrent tools is possible but not optimized in the base implementation [S1].

## Background
The method was introduced by Eric Ferley, Marie-Paule Cani, and Jean-Dominique Gascuel in their paper "Resolution Adaptive Volume Sculpting" [S1]. It addresses the limitation of recursive field update methods, which process depth-first and fail to maintain the coarse-to-fine consistency required for real-time interaction. The implementation was tested with POSIX threads, IRIX sprocs, and Windows threads, showing no significant performance differences [S1].

## Sources
[S1] Resolution Adaptive Volume Sculpting, https://doi.org/10.1006/gmod.2001.0558, fetched 2026-10-26
