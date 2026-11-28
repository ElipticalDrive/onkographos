# openvdb 13.0.0 (Major v13)

VFX Platform 2025 compatible build package for openvdb.

## Package Information

- **Package Name**: openvdb
- **Version**: 13.0.0
- **Major Version**: 13
- **Repository**: vfxplatform-2025/openvdb-13
- **Description**: OpenVDB: Sparse volume data structure and tools.

## Build Instructions

```bash
rez-build -i
```

## Package Structure

```
openvdb/
├── 13.0.0/
│   ├── package.py      # Rez package configuration
│   ├── rezbuild.py     # Build script
│   ├── get_source.sh   # Source download script (if applicable)
│   └── README.md       # This file
```

## Installation

When built with `install` target, installs to: `/core/Linux/APPZ/packages/openvdb/13.0.0`

## Version Strategy

This repository contains **Major Version 13** of openvdb. Different major versions are maintained in separate repositories:

- Major v13: `vfxplatform-2025/openvdb-13`

## VFX Platform 2025

This package is part of the VFX Platform 2025 initiative, ensuring compatibility across the VFX industry standard software stack.
