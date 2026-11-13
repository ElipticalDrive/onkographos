# Splat Cloud Loading

Also called: SPZ/PLY Parsing

## Overview
Splat Cloud Loading is an asynchronous file parsing method that loads a `SplatCloud` data structure from `.spz` or `.ply` files off the main thread [S1, S2, unofficial]. It solves the problem of blocking the UI during the loading of large 3D Gaussian Splat captures by returning `null` while parsing is in progress [S1, S2, unofficial]. It is distinguished from synchronous parsing methods by its non-blocking nature and its support for multiple compression formats (gzip and ZSTD) within a single pure-Kotlin implementation [S2, unofficial].

## How it works
**Inputs and Outputs:**
- Input: `fileLocation` (String) specifying the source of the splat data [S1, S2, unofficial].
- Output: `SplatCloud?` (nullable object). Returns `null` while loading or if the file cannot be read [S1, S2, unofficial]. Returns a non-empty `SplatCloud` instance upon successful completion [S1, unofficial].

**Data Structures:**
- `SplatCloud`: The resulting data structure representing the loaded splats. It contains a `count` property indicating the number of splats [S1, unofficial]. The structure must be non-empty upon successful return [S1, unofficial].

**Supported Formats:**
- `.spz` versions 2 and 3: Gzip compressed [S2, unofficial].
- `.spz` version 4: ZSTD streams (current Niantic format) [S2, unofficial].
- `.ply`: Point Cloud Library format [S1, S2, unofficial].

**Steps:**
1. The function receives a `fileLocation` string [S1, S2, unofficial].
2. The location is resolved. Supported schemes include:
   - Asset paths (e.g., `"splats/scan.spz"`) [S1, S2, unofficial].
   - Absolute file paths [S1, S2, unofficial].
   - URIs: `file://`, `content://`, `http://`, or `https://` [S1, S2, unofficial].
3. The file is read asynchronously off the main thread [S1, S2, unofficial].
4. While loading, the function returns `null` [S1, S2, unofficial].
5. If the file cannot be read or parsing fails, the function returns `null` [S2, unofficial].
6. Upon successful parsing, the binary data is decompressed (if compressed) and parsed into a `SplatCloud` object using pure Kotlin logic [S2, unofficial].
7. The `SplatCloud` object is returned [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# This is a conceptual representation of the Kotlin Composable function behavior.
# The actual implementation is in Kotlin and runs on the main thread as a Composable,
# but performs the heavy I/O and parsing off the main thread.

fun rememberSplatCloud(fileLocation: String) -> SplatCloud? {
    # fileLocation: String
    # Returns: SplatCloud? (null while loading or on error)
    
    # 1. Validate or prepare the file source based on fileLocation
    # Supported: asset path, absolute path, file://, content://, http(s)://
    
    # 2. Check if already loaded/cached (implementation detail)
    # If cached and valid, return the SplatCloud.
    
    # 3. If not loaded, initiate asynchronous loading
    # The actual parsing logic (SplatParser) is called off the main thread.
    
    # 4. While the background task is running, return null
    # This allows the UI to remain responsive.
    
    # 5. When the background task completes:
    #    a. If success: Parse bytes using SplatParser (auto-detects .spz v2-4 or .ply)
    #       - Decompress gzip (v2/v3) or ZSTD (v4) if necessary.
    #       - Construct SplatCloud object.
    #    b. If failure: Return null.
    
    # 6. Update state and return the SplatCloud or null.
    
    # Note: The specific threading and state management is handled by the 
    # Kotlin Compose framework and the SplatParser library.
    return null # Placeholder for the actual async result
}
```

**Parameters:**
- `fileLocation`: String. The source of the splat file. Can be an asset path, an absolute path, or a `file://`, `content://`, or `http(s)://` URI [S1, S2, unofficial].

**Edge Cases and Pitfalls:**
- The function returns `null` if the file cannot be read [S2, unofficial].
- The `SplatCloud` must be non-empty [S1, unofficial].
- Parsing is implemented in pure Kotlin, ensuring compatibility across all target platforms [S2, unofficial].

## Subtopics

## Operating characteristics
- **Time Complexity:** Not given in the sources. Depends on file size and compression ratio.
- **Memory Complexity:** Not given in the sources. Depends on the number of splats and the `SplatCloud` structure layout.
- **Performance:** The method is designed to run off the main thread to prevent UI blocking [S1, S2, unofficial].
- **Strengths:** Asynchronous loading prevents UI freezes; supports multiple compression formats (gzip, ZSTD) and file types (SPZ, PLY); pure Kotlin implementation ensures cross-platform compatibility [S2, unofficial].
- **Limitations:** Returns `null` during loading, requiring the caller to handle nullable states [S2, unofficial]. Clouds above 65535 splats are split into multiple nodes (implied by parent article context, but not explicitly detailed in this article's notes) [S1, unofficial].
- **When to use:** When loading large splat files in a UI environment where blocking the main thread is unacceptable.
- **When not to use:** When synchronous loading is required or when the file format is not supported (only .spz v2-4 and .ply are supported).

## Background
The method was introduced in the `sceneview` library to improve the user experience when loading 3D Gaussian Splat captures. It was added to allow loading `.spz` and `.ply` files off the main thread, returning `null` while loading [S1, S2, unofficial]. The implementation supports `.spz` versions 2, 3, and 4, with version 4 being the current Niantic format [S2, unofficial]. The parsing logic is provided by the `SplatParser` class in the `io.github.sceneview.core.splat` package, which uses pure Kotlin [S1, S2, unofficial].

## Sources
[S1] feat(sceneview): sort splats for the scene camera by default and add …, https://github.com/sceneview/sceneview/commit/e7e8fd42c0b46c8e9c62db902f6bb1e60474006f, fetched 2026-10-30 (unofficial)
[S2] feat(sceneview): sort splats for the scene camera by default and add …, https://github.com/sceneview/sceneview/commit/5f2a4dd1f683ddb444669be54b78ce7913b55b70, fetched 2026-10-30 (unofficial)
