# JavaVoxels
Data structures and file loaders for working with voxel data


This is a library project intended to work with voxels.

## What do I mean by voxels?

I don't intend to pass judgement on what a voxel is. It could just be color information, it could be a fireplace, it could be a planet in a universe. There's a reason this stuff is generic. The reason that I'm calling this data "voxel data" is because it's spatial in nature, and because meaningful differences in the Stuff in that space happen to coincide with the boundaries in a regular cubic lattice. So a voxel is whatever you want it to be, as long as it can be considered to be confined entirely to the cube at its location.

## What can I do with these voxels?

There's nothing here that will render voxels for you. Everything else is in-scope: loading, saving, various kinds of small and large scale edits. Remap, combine, and transform data. Cache and programmatically generate it. Noise functions are out-of-scope but have a look at libnoise-java which should work nicely with the cache/gen functions here.

## How do I include it?

Unfortunately, at this moment jitpack or git/gradle submodules are the best way to get voxels into your project. More options as we get further along.

## Can I use this project in my [commercial or non-commercial] software? Can I contribute?

Yes and yes. Please do. This project is MIT licensed and PRs are a gift to me.
