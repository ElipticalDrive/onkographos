# Paged Block Array

Also called: Block Array, Block-allocated Dynamic Array, Chunked Array, Paged Array

## Overview
The Paged Block Array is a dynamic array implementation that stores elements in fixed-size blocks (pages) to optimize memory allocation and cache locality, avoiding the need for contiguous reallocation of the entire array [S1, unofficial]. It differs from standard dynamic arrays by using fixed-size blocks instead of contiguous reallocation, reducing allocation overhead and fragmentation at the cost of slightly higher pointer indirection and fixed block sizes [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A `BlockArray` structure, an element to insert, and an index.
- Output: A pointer to the element, or void for modification operations.

**Data Structures:**
The implementation relies on three core structures. The element type is generic (`Element`), but the block size is fixed.

1.  **Block Structure**:
    ```c
    struct ElementBlock {
        Element Elements[N];
    };
    ```
    Where $N$ is the block size. For index-based arrays, $N=8$ [S1, unofficial]. For pointer-based arrays, $N=64$ [S1, unofficial].

2.  **Index Structure**:
    ```c
    struct BlockArrayIndex {
        umm Index; // Global linear index into the array
    };
    ```
    `umm` is an unsigned integer type sufficient to hold the total element count [S1, unofficial].

3.  **BlockArray Structure**:
    ```c
    struct BlockArray {
        ElementBlock **BlockPtrs; // Pointer to array of pointers to blocks
        u32 BlockCount;           // Number of allocated blocks
        u32 ElementCount;         // Number of valid elements
        MemoryArena *Memory;      // Memory allocator
    };
    ```

**Parameters:**
- `N`: Block size. Typically 8 for index types, 64 for pointer types [S1, unofficial].
- `Memory`: A memory arena used for allocation [S1, unofficial].

**Algorithms:**

1.  **GetPtr**: Retrieves a pointer to the element at a global linear index.
    - Calculate `BlockIndex = Index / N`.
    - Assert `BlockIndex < BlockCount`.
    - Retrieve `Block = BlockPtrs[BlockIndex]`.
    - Calculate `ElementIndex = Index % N`.
    - Return `Block->Elements + ElementIndex`.

2.  **Push**: Appends an element to the end of the array.
    - Check if `ElementCount == Capacity` (where `Capacity = BlockCount * N`).
    - If full, call `NewBlock`.
    - Call `Set` to place the element at `ElementCount`.
    - Increment `ElementCount`.

3.  **NewBlock**: Allocates a new block and updates the pointer array.
    - Allocate `NewBlock` of type `ElementBlock` from `Memory`.
    - Allocate `NewBlockPtrs` array of size `BlockCount + 1` from `Memory`.
    - Copy `BlockPtrs[0..BlockCount-1]` to `NewBlockPtrs[0..BlockCount-1]`.
    - Set `NewBlockPtrs[BlockCount] = NewBlock`.
    - Update `BlockPtrs = NewBlockPtrs`.
    - Increment `BlockCount`.

4.  **RemoveUnordered**: Removes an element at `Index` by swapping it with the last element.
    - Get `LastI = LastIndex` (where `LastIndex = ElementCount - 1`).
    - Assert `Index <= LastI`.
    - Get `LastElement = GetPtr(Array, LastI)`.
    - Set `Array[Index] = LastElement`.
    - Decrement `ElementCount`.

5.  **RemoveOrdered**: Removes an element at `Index` by shifting subsequent elements.
    - Assert `Index < ElementCount`.
    - Iterate from `Index` to `LastIndex`.
    - Shift each element down by one position.
    - Decrement `ElementCount`.

6.  **Insert**: Inserts an element at `Index`, shifting subsequent elements.
    - Assert `Index <= LastIndex`.
    - Call `Push` to allocate space and get a pointer to the new last element (`Prev`).
    - Iterate backwards from `LastIndex` to `Index`.
    - Shift each element up by one position.
    - Set the new element at `Index`.

7.  **Pop**: Removes and returns the last element.
    - If `ElementCount > 0`, decrement `ElementCount` and return pointer to last element.
    - Else return empty/null.

8.  **ClearList**: Resets the array.
    - Iterate over all elements and set them to empty/zero.
    - Set `ElementCount = 0`.

9.  **Find**: Searches for an element.
    - Iterate over all elements.
    - Compare each with `Query` using `AreEqual`.
    - Return `Index` if match found, else `INVALID_BLOCK_ARRAY_INDEX`.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
N = 8  # Block size for index-based arrays

# Data Structures
class BlockArrayIndex:
    def __init__(self, index):
        self.Index = index

class ElementBlock:
    def __init__(self):
        self.Elements = [None] * N

class BlockArray:
    def __init__(self, memory_arena):
        self.BlockPtrs = []
        self.BlockCount = 0
        self.ElementCount = 0
        self.Memory = memory_arena

    def Capacity(self):
        return self.BlockCount * N

    def AtElements(self):
        return self.ElementCount

    def LastIndex(self):
        if self.ElementCount == 0:
            return BlockArrayIndex(0)
        return BlockArrayIndex(self.ElementCount - 1)

    def GetBlock(self, index):
        block_index = index // N
        assert block_index < self.BlockCount
        return self.BlockPtrs[block_index]

    def GetPtr(self, index):
        assert self.BlockPtrs
        assert index < self.ElementCount
        assert index < self.Capacity()
        block = self.GetBlock(index)
        element_index = index % N
        return block.Elements[element_index]

    def Set(self, element, index):
        assert self.BlockPtrs
        assert index < self.Capacity()
        block = self.GetBlock(index)
        element_index = index % N
        block.Elements[element_index] = element

    def NewBlock(self):
        new_block = ElementBlock()
        # Allocate new pointer array
        new_block_ptrs = [None] * (self.BlockCount + 1)
        # Copy existing pointers
        for i in range(self.BlockCount):
            new_block_ptrs[i] = self.BlockPtrs[i]
        # Add new block
        new_block_ptrs[self.BlockCount] = new_block
        # Update array
        self.BlockPtrs = new_block_ptrs
        self.BlockCount += 1

    def Push(self, element):
        assert self.Memory
        if self.AtElements() == self.Capacity():
            self.NewBlock()
        self.Set(element, self.AtElements())
        self.ElementCount += 1
        return self.GetPtr(self.AtElements() - 1)

    def RemoveUnordered(self, index):
        last_i = self.LastIndex()
        assert index <= last_i.Index
        last_element = self.GetPtr(last_i.Index)
        self.Set(last_element, index)
        self.ElementCount -= 1

    def RemoveOrdered(self, index_to_remove):
        assert index_to_remove < self.ElementCount
        prev = None
        for i in range(index_to_remove, self.ElementCount):
            e = self.GetPtr(i)
            if prev is not None:
                prev = e
            # Shift logic: move e to prev position
            # Note: In the source, Prev is updated to E, and *Prev = *E happens in next iteration
            # This effectively shifts elements down
            if i > index_to_remove:
                self.Set(self.GetPtr(i-1), i-1) # Simplified shift
        self.ElementCount -= 1

    def Insert(self, index, element):
        assert index <= self.LastIndex().Index
        assert self.Memory
        # Allocate space
        self.Push(None) # Push a default element
        last = self.LastIndex()
        # Shift elements up
        for i in range(last.Index, index, -1):
            e = self.GetPtr(i)
            self.Set(e, i-1)
        self.Set(element, index)

    def Pop(self):
        if self.ElementCount > 0:
            self.ElementCount -= 1
            return self.GetPtr(self.ElementCount)
        return None

    def ClearList(self):
        for i in range(self.ElementCount):
            self.Set(None, i)
        self.ElementCount = 0

    def Find(self, query):
        for i in range(self.ElementCount):
            if self.GetPtr(i) == query: # Assuming AreEqual logic
                return i
        return -1
```

**Edge Cases and Pitfalls:**
- `NewBlock` reallocates the entire `BlockPtrs` array. This invalidates any existing pointers to the `BlockPtrs` array itself, though pointers to individual blocks remain valid [S1, unofficial].
- Empty array pop: `TryGetPtr` checks `LastIndex` before accessing. Returns empty result if `ElementCount` is 0, preventing out-of-bounds access [S1, unofficial].

## Operating characteristics
- **Time Complexity**:
    - `Push`: O(1) amortized. `NewBlock` is O(N) where N is the number of blocks due to pointer array reallocation and copy, but happens infrequently [S1, unofficial].
    - `GetPtr`: O(1).
    - `RemoveUnordered`: O(1).
    - `RemoveOrdered`: O(K) where K is the number of elements after the removed index.
    - `Insert`: O(K) where K is the number of elements after the insertion index.
- **Memory Complexity**: O(B * N) where B is the number of blocks. There is some overhead for the `BlockPtrs` array.
- **Strengths**: Reduces allocation overhead and fragmentation compared to contiguous dynamic arrays. Good cache locality for sequential access within blocks.
- **Limitations**: Fixed block sizes may lead to internal fragmentation if element sizes are not multiples of block size or if not all slots are used. Pointer indirection adds slight overhead.
- **When to use**: When the number of elements is dynamic and large, and allocation overhead is a concern. Suitable for voxel stacks or similar data structures where elements are added/removed frequently.
- **When not to use**: When random access to the entire array is the primary operation and memory overhead of pointer indirection is unacceptable. When the element count is small and known in advance.

## Background
The Paged Block Array is described in the context of the Bonsai voxel engine, a project by Scallyw4g [S1, unofficial]. It is used for managing dynamic arrays of voxel-related data, such as chunk completion callbacks and work queue entries. The design choice reflects a need for efficient memory management in a voxel engine where data structures grow and shrink dynamically.

## Sources
[S1] Bonsai: A Voxel Engine, from scratch, https://github.com/scallyw4g/bonsai, fetched 2026-10-26 (unofficial)
