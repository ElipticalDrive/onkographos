# MetaVolume Data Loader and Time-Step Cache

Also called: MetaVolume::readVol, MetaVolume::swapTStep

## Overview
A data loading and caching system for volumetric datasets composed of multiple sub-volumes with time steps. It parses a configuration file, reads raw binary volume data, handles data type conversion and endian swapping, and caches previous time steps to avoid re-reading from disk [S1, unofficial]. It builds on [Raw Volume Data Reader](2259-raw-volume-data-reader.md) by adding multi-subvolume support and temporal caching.

## How it works
**Inputs and Outputs:**
- Input: Configuration file path, sub-volume index, time step index, cache size.
- Output: Pointer to volume data array (`currentData`) and gradient array (`currentGrad`) for the requested time step.

**Data Structures:**
- `volumes[subv]`: Struct storing dimensions `x, y, z` (int) and data type.
- `oldData`: Array of pointers to cached volume data. Size: `tstepCache`.
- `oldGrad`: Array of pointers to cached gradient data. Size: `tstepCache`.
- `oldTimeStep`: Integer tracking the time step index of the most recently accessed cache slot. Initialized to -1 [S1, unofficial].
- `oldTSpos`: Integer index of the current cache slot. Initialized to 0 [S1, unofficial].

**Steps:**
1. **Parse Configuration:** Read the configuration file line by line. Identify keywords: `"Size int"`, `"Size float"`, `"Pos int"`, `"Pos float"`. Extract `x, y, z` values using `strtok` and `atoi`/`atof`. Store in `volumes[subv]` struct. Handle syntax errors if values are missing [S1, unofficial].
2. **Initialize Cache:** Allocate `oldData` and `oldGrad` arrays of size `tstepCache`. Initialize `oldTimeStep` to -1 and `oldTSpos` to 0 [S1, unofficial].
3. **Construct File Path:** Determine the file path for the requested sub-volume and time step. If the `append` flag is true, the format is `path.timestep.volNum` (e.g., `%s.%04d.%02d`). Else, use `path` [S1, unofficial].
4. **Check Cache:** If the requested time step matches `oldTimeStep`, return the cached data pointers from `oldData[oldTSpos]` and `oldGrad[oldTSpos]` [S1, unofficial].
5. **Load Data:** If not in cache, read volume data for the specific sub-volume and time step.
    - If `dataType == _DOUBLE`, read directly into `dbData[0]`. Verify byte count. Set `currentData` [S1, unofficial].
    - For other data types, perform data type conversion for floating-point data if necessary [S1, unofficial].
6. **Update Cache:** Store the new data pointers in `oldData[oldTSpos]` and `oldGrad[oldTSpos]`. Update `oldTimeStep` to the current time step. Increment `oldTSpos` modulo `tstepCache` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def load_volume(subv, timestep, tstepCache, append, dataType, config_path):
    # Step 1: Parse configuration (simplified)
    x, y, z = parse_config(config_path, subv)
    
    # Step 2: Initialize cache (simplified, assuming global state)
    if not cache_initialized:
        oldData = [None] * tstepCache
        oldGrad = [None] * tstepCache
        oldTimeStep = -1
        oldTSpos = 0
        cache_initialized = True
    
    # Step 3: Construct file path
    if append:
        filepath = f"{config_path}.{timestep:04d}.{subv:02d}"
    else:
        filepath = config_path
        
    # Step 4: Check cache
    if oldTimeStep == timestep:
        return oldData[oldTSpos], oldGrad[oldTSpos]
    
    # Step 5: Load data
    # Note: Actual reading depends on dataType and endianness handling from parent
    if dataType == _DOUBLE:
        data = read_double_from_disk(filepath, x * y * z)
    else:
        data = read_raw_from_disk(filepath, x * y * z, dataType)
        # Convert if necessary [S1, unofficial]
        
    # Step 6: Update cache
    oldData[oldTSpos] = data
    oldGrad[oldTSpos] = compute_gradients(data, x, y, z) # Gradient computation assumed or handled elsewhere
    oldTimeStep = timestep
    oldTSpos = (oldTSpos + 1) % tstepCache
    
    return oldData[oldTSpos], oldGrad[oldTSpos]
```

**Parameters:**
- `tstepCache`: Number of time steps to keep in memory cache [S1, unofficial].
- `append`: Boolean flag. If true, sub-volume file names include time step and volume number suffixes [S1, unofficial].
- `dataType`: Enum for data type. Values: `_UCHAR`, `_USHORT`, `_SHORT`, `_UINT`, `_INT`, `_FLOAT`, `_DOUBLE` [S1, unofficial].

**Edge Cases and Pitfalls:**
- Precision: Reading `_DOUBLE` directly into `dbData[0]` requires careful byte count verification [S1, unofficial].
- Boundaries: Syntax errors in configuration files must be handled [S1, unofficial].
- Degenerate Input: Missing values in configuration files should trigger error handling [S1, unofficial].

## Operating characteristics
- **Time Complexity:** O(1) for cached reads, O(N) for disk reads where N is the number of voxels.
- **Memory Complexity:** O(tstepCache * N) for storing cached data.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales with the number of sub-volumes and time steps.
- **Strengths:** Avoids disk I/O and allocation overhead for repeated time steps [S1, unofficial].
- **Limitations:** Limited by cache size.
- **When to use:** When accessing the same time steps multiple times.
- **When not:** When memory is constrained or time steps are accessed sequentially without repetition.

## Background
Origin: Not given in the sources.
History and development: Not given in the sources.
Where it is used: Not given in the sources.

## Sources
[S1] ouyting/simian-spacemonkey, https://github.com/ouyting/simian-spacemonkey, fetched 2026-10-27 (unofficial)
