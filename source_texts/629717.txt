# VoxFormat

## Overview
VoxFormat is a part of the VOX4U plugin for Unreal Engine 4, which is used to import MagicaVoxel VOX format files. It provides the functionality to read and interpret the VOX format.

## Installation
VoxFormat is included as part of the VOX4U plugin. To install the VOX4U plugin, please follow the instructions in the main [VOX4U README](https://github.com/mik14a/VOX4U).

## Usage
VoxFormat is used internally by the VOX4U plugin to handle the reading and interpretation of VOX format files. It provides the `FVox` structure which is used to represent the data in a VOX file. The `FVox::Read` function is used to create an `FVox` instance from a data buffer.

Here is a basic example of how VoxFormat might be used within the VOX4U plugin:

```cpp
// Create a buffer to hold the VOX data
const void Data;
int64 Size;

// Fill the buffer with VOX data...

// Create an FVox instance from the data
FVox Vox = ReadVox(Data, Size);
```

Please note that VoxFormat is intended to be used as part of the VOX4U plugin and may not work correctly if used in isolation.

## License
VoxFormat is licensed under the MIT License. For more details, please refer to the [LICENSE](https://github.com/mik14a/VOX4U/blob/master/LICENSE) file.

## Contact
For any queries or issues, please raise them on the [VOX4U GitHub repository](https://github.com/mik14a/VOX4U).
