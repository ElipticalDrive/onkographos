# OpenVDB Volume Data Source

Also called: OpenVDB Support for Mitsuba, VdbDataSource

## Overview
The OpenVDB Volume Data Source is a plugin for the Mitsuba renderer that enables ray marching through volumetric data stored in the OpenVDB sparse voxel format [S1, unofficial]. It implements the `VolumeDataSource` interface, providing scalar field lookups via trilinear interpolation and automatic bounding box calculation [S1, unofficial]. It is distinguished from generic import methods by its integration into the rendering pipeline's caching system and its specific handling of OpenVDB's coordinate transforms and metadata [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:** `filename` (string, path to OpenVDB file), `fieldname` (string, name of the grid within the file), `customStepSize` (float, optional override for sampling step size) [S1, unofficial].
*   **Outputs:** Scalar float value at a given 3D point, bounding box of the volume, step size for ray marching, and maximum value in the grid [S1, unofficial].

**Data Structures**
The method relies on a global static cache to avoid reloading files:
1.  `VdbGridSet`: A struct containing `std::map<std::string, openvdb::GridBase::Ptr> m_grids`, mapping grid names to OpenVDB grid pointers [S1, unofficial].
2.  `VdbGridPool`: A struct containing `std::map<std::string, VdbGridSet> m_gridSets`, mapping filenames to their respective `VdbGridSet` [S1, unofficial].
3.  `s_vdbGridPool`: A static global instance of `VdbGridPool` [S1, unofficial].

**Numbered Steps**

1.  **Initialization:** Call `openvdb::initialize()` [S1, unofficial]. Read `filename`, `fieldname`, and `customStepSize` from properties [S1, unofficial]. If `customStepSize` is not provided or is $\le 0.0$, it defaults to $0.0$ [S1, unofficial].
2.  **Cache Check:** Check if `filename` exists in `s_vdbGridPool.m_gridSets` [S1, unofficial].
    *   If not present, call `open()` to load the file and populate the cache [S1, unofficial].
    *   If present, skip loading [S1, unofficial].
3.  **Field Validation:** After ensuring the file is loaded, check if `fieldname` exists in the `VdbGridSet` for that filename [S1, unofficial].
    *   If the field is missing, log an error and fail initialization [S1, unofficial].
4.  **File Opening (`open()`):**
    *   Create an `openvdb::io::File` object with `filename` [S1, unofficial].
    *   Call `file->open()` [S1, unofficial].
    *   Read all grid metadata using `file->readAllGridMetadata()` [S1, unofficial].
    *   Iterate through all grids to calculate the global bounding box:
        *   Initialize `bbMin` to $(\infty, \infty, \infty)$ and `bbMax` to $(-\infty, -\infty, -\infty)$ [S1, unofficial].
        *   For each grid, retrieve `META_FILE_BBOX_MIN` and `META_FILE_BBOX_MAX` as `openvdb::Vec3i` [S1, unofficial].
        *   Convert these integer indices to world coordinates using `grid->indexToWorld()` [S1, unofficial].
        *   Update `bbMin` using `minComponent` and `bbMax` using `maxComponent` across all grids [S1, unofficial].
    *   Store the resulting `bbMin` and `bbMax` in the volume's `m_aabb` [S1, unofficial].
    *   Insert the grids into `s_vdbGridPool` under the `filename` key [S1, unofficial].
5.  **Float Lookup (`lookupFloat`):**
    *   Retrieve the grid from the cache using `filename` and `fieldname` [S1, unofficial].
    *   Cast the grid to `openvdb::FloatGrid` [S1, unofficial].
    *   Create a `GridSampler` using the grid's tree and transform, configured with `openvdb::tools::BoxSampler` (trilinear interpolation) [S1, unofficial].
    *   Sample the world-space point $p$ using `interpolator.wsSample(openvdb::math::Vec3d(p.x, p.y, p.z))` [S1, unofficial].
6.  **Step Size Calculation (`getStepSize`):**
    *   Default `stepSize` is $1.0$ [S1, unofficial].
    *   If `m_customStepSize > 0.0`, use `m_customStepSize` [S1, unofficial].
    *   Otherwise, retrieve the voxel size from the grid's transform: `voxelSize = grid->constTransform().voxelSize()` [S1, unofficial].
    *   Calculate `stepSize` as $0.5 \times \min(\text{voxelSize}.x(), \text{voxelSize}.y(), \text{voxelSize}.z())$ [S1, unofficial].
7.  **Max Value Calculation (`getMaximumFloatValue`):**
    *   Default `valMax` is $-1.0$ [S1, unofficial].
    *   If the grid exists, cast to `FloatGrid` and call `evalMinMax(valMin, valMax)` [S1, unofficial].
    *   Return `valMax` [S1, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

# Global cache
s_vdbGridPool = {}  # Map: filename -> VdbGridSet

class VdbGridSet:
    def __init__(self):
        self.m_grids = {}  # Map: grid_name -> GridBase_ptr

class Vec3d:
    def __init__(self, x, y, z):
        self.x = x
        self.y = y
        self.z = z

def min_component(a, b):
    return Vec3d(min(a.x, b.x), min(a.y, b.y), min(a.z, b.z))

def max_component(a, b):
    return Vec3d(max(a.x, b.x), max(a.y, b.y), max(a.z, b.z))

def openvdb_initialize():
    pass  # Initializes OpenVDB library

def cast_float_grid(grid_ptr):
    return grid_ptr  # Casts to FloatGrid type

def create_sampler(tree, transform):
    return GridSampler(tree, transform)

class GridSampler:
    def __init__(self, tree, transform):
        self.tree = tree
        self.transform = transform
    def ws_sample(self, point):
        # Uses openvdb::tools::BoxSampler (trilinear interpolation)
        return sample_trilinear(self.tree, self.transform, point)

class OpenVDBFile:
    def __init__(self, filename):
        self.filename = filename
    def open(self):
        return True  # Simulates file->open()
    def readAllGridMetadata(self):
        return []  # Returns list of grid metadata objects

class VdbDataSource:
    def __init__(self, props):
        openvdb_initialize()
        self.m_filename = props.get("filename", "")
        self.m_fieldname = props.get("fieldname", "")
        self.m_customStepSize = props.get("customStepSize", 0.0)

        if self.m_filename not in s_vdbGridPool:
            self.open()
        
        if self.m_fieldname not in s_vdbGridPool[self.m_filename].m_grids:
            raise RuntimeError(f"Can't get field {self.m_fieldname} from {self.m_filename}")

    def open(self):
        file = OpenVDBFile(self.m_filename)
        if not file.open():
            raise RuntimeError(f"Can't open file {self.m_filename}")
        
        s_vdbGridPool[self.m_filename] = VdbGridSet()
        grid_set = s_vdbGridPool[self.m_filename]
        
        bbMin = Vec3d(math.inf, math.inf, math.inf)
        bbMax = Vec3d(-math.inf, -math.inf, -math.inf)
        
        grids = file.readAllGridMetadata()
        for grid in grids:
            bbMinI = grid.metaValue("META_FILE_BBOX_MIN")
            bbMaxI = grid.metaValue("META_FILE_BBOX_MAX")
            
            bbMin = min_component(bbMin, grid.indexToWorld(bbMinI))
            bbMax = max_component(bbMax, grid.indexToWorld(bbMaxI))
            
            grid_set.m_grids[grid.name()] = grid

        self.m_aabb.min = bbMin
        self.m_aabb.max = bbMax

    def lookupFloat(self, p):
        if self.m_filename not in s_vdbGridPool:
            return 0.0
        
        grid_ptr = s_vdbGridPool[self.m_filename].m_grids.get(self.m_fieldname)
        if not grid_ptr:
            return 0.0
            
        float_grid = cast_float_grid(grid_ptr)
        interpolator = create_sampler(float_grid.constTree(), float_grid.transform())
        return interpolator.ws_sample(Vec3d(p.x, p.y, p.z))

    def getStepSize(self):
        if self.m_customStepSize > 0.0:
            return self.m_customStepSize
            
        if self.m_filename in s_vdbGridPool:
            grid_ptr = s_vdbGridPool[self.m_filename].m_grids.get(self.m_fieldname)
            if grid_ptr:
                voxel_size = grid_ptr.constTransform().voxelSize()
                min_voxel = min(voxel_size.x, voxel_size.y, voxel_size.z)
                return 0.5 * min_voxel
        return 1.0

    def getMaximumFloatValue(self):
        valMax = -1.0
        if self.m_filename in s_vdbGridPool:
            grid_ptr = s_vdbGridPool[self.m_filename].m_grids.get(self.m_fieldname)
            if grid_ptr:
                float_grid = cast_float_grid(grid_ptr)
                valMin = 0.0
                float_grid.evalMinMax(valMin, valMax)
        return valMax

    def supportsFloatLookups(self):
        return True

    def supportsVectorLookups(self):
        return False
```

**Parameters**
*   `filename`: Path to the OpenVDB file [S1, unofficial].
*   `fieldname`: Name of the specific grid/field to render [S1, unofficial].
*   `customStepSize`: Optional float. If $> 0.0$, overrides automatic step size calculation. Default $0.0$ [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Missing Field:** If `fieldname` does not exist in the loaded file, the plugin logs an `EError` and fails [S1, unofficial].
*   **Missing File:** If the file cannot be opened, the plugin logs an `EError` [S1, unofficial].
*   **Vector Data:** The plugin explicitly returns `false` for `supportsVectorLookups()`, meaning it only supports scalar (float) volume data [S1, unofficial].
*   **Interpolation:** Uses `BoxSampler` (trilinear interpolation). It does not support nearest-neighbor or higher-order splines [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Lookup is $O(1)$ relative to the number of grids, but depends on the depth of the OpenVDB tree and the interpolation cost [S1, unofficial]. Bounding box calculation is $O(N)$ where $N$ is the number of grids in the file [S1, unofficial].
*   **Memory Complexity:** Depends on the OpenVDB file size [S1, unofficial]. The global cache `s_vdbGridPool` retains loaded files in memory, which can lead to high memory usage if many large files are loaded [S1, unofficial].
*   **Strengths:** Efficient sparse data handling via OpenVDB; automatic bounding box and step size calculation; caching prevents redundant file I/O [S1, unofficial].
*   **Limitations:** Only supports scalar float data; uses trilinear interpolation which may be insufficient for high-frequency details compared to higher-order methods; no built-in mechanism to unload cached grids [S1, unofficial].
*   **When to use:** When rendering volumetric data stored in OpenVDB format within Mitsuba and scalar interpolation is sufficient [S1, unofficial].
*   **When not to use:** When vector fields are required or when higher-order interpolation is needed [S1, unofficial].

## Background
The OpenVDB Volume Data Source was developed by Bo Zhou and released as part of the `mitsuba-vdb` plugin for the Mitsuba renderer [S1, unofficial]. It was created to bridge the gap between the sparse voxel data structures used in visual effects (via OpenVDB) and the ray marching capabilities of Mitsuba [S1, unofficial]. It requires Boost 1.53, TBB, IlmBase, and OpenVDB to compile [S1, unofficial].

## Sources
[S1] Mitsuba-VDB plugin that can be found on Github, https://github.com/zhoub/mitsuba-vdb, fetched 2026-10-30 (unofficial)
