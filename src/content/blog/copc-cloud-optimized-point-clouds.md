---
author: Obed Macallums
pubDatetime: 2026-08-16T10:00:00Z
title: "COPC: Cloud Optimized Point Clouds for Streaming LiDAR"
slug: copc-cloud-optimized-point-clouds
featured: false
draft: false
tags:
  - copc
  - point-cloud
  - lidar
  - geospatial
  - pdal
  - python
  - las
description:
    A practical guide to COPC (Cloud Optimized Point Cloud), the LAZ 1.4 based format that lets you stream massive LiDAR datasets over HTTP without downloading them. Learn its internal structure, how to create COPC files with PDAL and Untwine, how to query them remotely from Python, and how it compares to Entwine's EPT format.
---

You have a 50 GB **.LAZ** file sitting in an S3 bucket. A user wants to inspect one city block of it. With a plain LAZ file there is no way around it: you download all 50 GB, decompress it, and throw away 99.99% of the points. **COPC** (Cloud Optimized Point Cloud) solves exactly this problem, and it does so without inventing a new format — a COPC file is still a perfectly valid LAZ file. In this article we look at how it works internally, how to produce and consume COPC files with PDAL and Python, and how it relates to Entwine's older EPT format.

## Table of Contents

## The Problem with Point Clouds in the Cloud

Formats like [.LAS and .LAZ](/posts/las-laz-format) were designed when point clouds lived on local disks. LAZ compresses beautifully — often 80% smaller than LAS — but compression alone does not make a file *usable* over a network. The bytes of a conventional LAZ file are laid out in whatever order the sensor or the processing software produced them, with no relationship between a point's position in the file and its position in space.

That has two consequences:

- **No spatial access**: to find the points inside a bounding box you must read every point.
- **No level of detail**: to draw a coarse overview of the dataset you must still read every point.

The raster world solved this years ago with **COG** (Cloud Optimized GeoTIFF): keep the same format, reorganize the bytes internally, add an index, and let clients fetch only the byte ranges they need using HTTP range requests. COPC is that same idea applied to LiDAR.

## What Is COPC?

**COPC** is an open specification published in 2021 by [Hobu, Inc.](https://hobu.co) — the same group behind [PDAL](/posts/pdal-point-clouds). Its definition is remarkably compact:

> A COPC file is a LAZ 1.4 file that stores point data organized in a clustered octree, described by a VLR in the file header.

Two properties fall out of that definition, and both matter:

1. **It is still a LAZ file.** Any reader that can handle variably-chunked LAZ 1.4 can read a `.copc.laz` file sequentially and get all the points, completely ignoring the octree. There is no fallback path to maintain, no second copy of your data.
2. **It is a single file.** The index lives inside the file, not beside it. One object in a bucket, one URL, one thing to version and cache.

Because the octree index records the exact byte offset and size of every node, a client that *does* understand COPC can issue HTTP range requests for just the nodes covering the area and resolution it cares about. No tile server, no database, no preprocessing service — a static file behind plain HTTP is enough.

## Anatomy of a COPC File

The specification defines three hard requirements that distinguish an organized COPC file from an ordinary LAZ 1.4 file.

### Point Data Record Format 6, 7, or 8

A COPC file **must** contain only ASPRS LAS Point Data Record Formats **6, 7, or 8**. These are the LAS 1.4 formats — PDRF 6 is the base format, 7 adds RGB, and 8 adds RGB plus near-infrared. The older PDRFs (0–5) are not allowed, which keeps readers simple and guarantees modern features like 256 classification values and extended return numbering.

### The `info` VLR

| User ID | Record ID | Size |
| ------- | --------- | ---- |
| `copc`  | `1`       | 160 bytes |

The `info` VLR **must** exist and **must** be the very first VLR in the file, beginning at byte offset **375** — immediately after the LAS 1.4 header. That fixed position is deliberate: a client can retrieve the header and the entire index descriptor in a single small range request before deciding what else to fetch.

Its 160 bytes describe the octree:

```c
struct CopcInfo
{
  // Actual (unscaled) coordinates of the center of the octree
  double center_x;
  double center_y;
  double center_z;

  // Perpendicular distance from the center to any side of the root node
  double halfsize;

  // Space between points at the root node.
  // This value is halved at each octree level
  double spacing;

  // File offset to the first hierarchy page, and its size in bytes
  uint64_t root_hier_offset;
  uint64_t root_hier_size;

  // Minimum and maximum of GPSTime
  double gpstime_minimum;
  double gpstime_maximum;

  // Must be 0
  uint64_t reserved[11];
};
```

The `spacing` field is what makes level-of-detail queries possible. The root node holds a sparse sample of the whole dataset with points roughly `spacing` units apart; each level down halves that distance. So the resolution available at level *n* is `spacing / 2ⁿ`, and a client that wants 1-metre resolution simply stops descending once it reaches that level.

### The `hierarchy` VLR

| User ID | Record ID |
| ------- | --------- |
| `copc`  | `1000`    |

The `hierarchy` VLR **must** exist too. It stores the octree itself as one or more *pages*, so that large hierarchies can be fetched incrementally rather than all at once. Each node is addressed by a `VoxelKey`:

```c
struct VoxelKey
{
  // A value < 0 indicates an invalid VoxelKey
  int32_t level;
  int32_t x;
  int32_t y;
  int32_t z;
}
```

And each entry in a hierarchy page is a fixed 32 bytes:

```c
struct Entry
{
  // Octree key of the data to which this entry corresponds
  VoxelKey key;

  // Absolute offset to the data chunk if pointCount > 0.
  // Absolute offset to a child hierarchy page if pointCount is -1.
  // 0 if pointCount is 0.
  uint64_t offset;

  // Size of the data chunk in bytes (compressed) if pointCount > 0.
  // Size of the hierarchy page if pointCount is -1.
  // 0 if pointCount is 0.
  int32_t byteSize;

  // If > 0, the number of points in the data chunk.
  // If -1, this node's information lives in another hierarchy page.
  // If 0, no point data exists for this key (children may still have data).
  int32_t pointCount;
}
```

Since every entry is exactly 32 bytes, the number of entries in a page is just `page_size / 32`. The `pointCount == -1` case is the mechanism for paging: instead of a chunk of points, the offset points at another hierarchy page, which the reader fetches only if it needs to descend into that branch.

Put together, the read path is: fetch bytes 0–535 to get the header and `info` VLR → fetch the root hierarchy page → walk the octree in memory, discarding nodes outside your bounds or below your target resolution → issue range requests for exactly the surviving chunks → decompress each chunk with LASzip.

## Creating COPC Files

### With PDAL

If you already use PDAL pipelines, producing COPC is a one-line change: swap `writers.las` for `writers.copc`.

```json
[
  "input.laz",
  {
    "type": "writers.copc",
    "filename": "output.copc.laz"
  }
]
```

The writer accepts multiple inputs, so you can merge a directory of tiles into a single COPC file in one pass:

```json
[
  "tile_001.laz",
  "tile_002.laz",
  "tile_003.laz",
  {
    "type": "writers.copc",
    "filename": "merged.copc.laz"
  }
]
```

Two options are worth knowing. `forward` preserves header fields from the source file — importantly the scale and offset values, which you almost always want to keep so precision is not silently degraded:

```json
{
  "type": "writers.copc",
  "filename": "output.copc.laz",
  "forward": "all"
}
```

And `a_srs` sets the spatial reference on the output if the input lacks one.

Remember that COPC requires PDRF 6, 7, or 8. If your source is an older LAS 1.2 file with PDRF 3, the writer handles the conversion, but attributes that only exist in the newer formats will be defaulted.

You can chain any of the filters covered in the [PDAL article](/posts/pdal-point-clouds) before writing — a realistic production pipeline usually cleans and reprojects before indexing:

```json
{
  "pipeline": [
    {
      "type": "readers.las",
      "filename": "raw_scan.laz"
    },
    {
      "type": "filters.reprojection",
      "out_srs": "EPSG:3857"
    },
    {
      "type": "filters.outlier",
      "method": "statistical",
      "mean_k": 8,
      "multiplier": 2.0
    },
    {
      "type": "filters.range",
      "limits": "Classification![7:7]"
    },
    {
      "type": "writers.copc",
      "filename": "web_ready.copc.laz",
      "forward": "all"
    }
  ]
}
```

Reprojecting to **EPSG:3857** is a common choice here specifically because most web viewers expect Web Mercator.

### With Untwine

PDAL's writer builds the octree in memory, which becomes a problem as datasets grow. For genuinely massive builds — hundreds of millions to billions of points — Hobu provides [Untwine](https://github.com/hobuinc/untwine), a dedicated out-of-core COPC builder that can index anything PDAL can read:

```bash
conda install -c conda-forge untwine

untwine -i some_directory_of_input_files -o output.copc.laz
```

Untwine is licensed under GPLv3, which is worth checking against your project's requirements. As a rule of thumb: reach for `writers.copc` when the job fits comfortably in RAM and you want it inside an existing pipeline, and for Untwine when it does not.

### Inspecting and Validating

`pdal info` works on COPC files like any other, and the metadata will show the COPC VLRs:

```bash
pdal info output.copc.laz
pdal info --stats output.copc.laz
```

The quickest visual check is [viewer.copc.io](https://viewer.copc.io) — drag the file onto the page, or point it at a URL, and it renders directly in the browser.

## Reading COPC Remotely

This is where the format earns its name. Both of the tools below fetch **only** the bytes they need; neither downloads the file.

### With PDAL

`readers.copc` accepts remote paths — HTTP, AWS, Google, Azure, and Dropbox:

```json
[
  {
    "type": "readers.copc",
    "filename": "https://s3.amazonaws.com/hobu-lidar/autzen-classified.copc.laz",
    "bounds": "([636800, 637200], [850700, 851100])"
  },
  {
    "type": "writers.las",
    "filename": "subset.laz"
  }
]
```

That pipeline pulls one small neighbourhood out of a remote dataset and writes it locally. The options that matter for remote reads:

- **`bounds`** — the extent to select, in 2 or 3 dimensions. Omitted means the whole dataset.
- **`polygon`** — a WKT clipping polygon, optionally followed by `/` and a spatial reference. Can be specified more than once, in which case the polygons are treated as a single multipolygon.
- **`resolution`** — limit which pyramid levels get fetched, in the units of the data. This is the level-of-detail knob: set it to `2.0` and the reader stops descending once points are about 2 units apart. Default is no limit.
- **`requests`** — number of worker threads issuing HTTP requests. Default is `15`. More is not automatically better: a fast connection can fetch data faster than PDAL can process it, which just inflates memory use.

Combining `resolution` with `bounds` is the common case for interactive work — a coarse look at a specific area costs a handful of range requests:

```json
{
  "type": "readers.copc",
  "filename": "https://example.com/city.copc.laz",
  "bounds": "([500000, 501000], [4500000, 4501000])",
  "resolution": 2.0
}
```

### With laspy

[laspy](/posts/las-laz-format) ships a dedicated `CopcReader`. It requires the **lazrs** backend, and the `requests` package for HTTP sources:

```bash
pip install laspy[lazrs] requests
```

Opening a remote file and querying a spatial subset:

```python
import numpy as np
from laspy import Bounds, CopcReader

url = "https://s3.amazonaws.com/hobu-lidar/autzen-classified.copc.laz"

with CopcReader.open(url) as reader:
    # The header is available immediately — only a few hundred bytes
    # have been transferred at this point
    print(f"Total points in file: {reader.header.point_count}")
    print(f"Bounds: {reader.header.mins} to {reader.header.maxs}")

    # Build a query covering the lower-left quadrant
    sizes = reader.header.maxs - reader.header.mins
    query_bounds = Bounds(
        mins=reader.header.mins,
        maxs=reader.header.mins + sizes / 2,
    )
    # Keep the full vertical range
    query_bounds.maxs[2] = reader.header.maxs[2]

    points = reader.query(query_bounds)
    print(f"Points fetched: {len(points)}")
    print(f"That is {len(points) / reader.header.point_count * 100:.2f}% of the file")
```

`query()` takes three optional arguments, and each of them reduces how many bytes cross the network:

```python
# Spatial subset only
points = reader.query(bounds=query_bounds)

# Spatial subset at a target resolution (in data units)
points = reader.query(bounds=query_bounds, resolution=1.0)

# Explicit octree levels — level 0 is the coarsest overview
points = reader.query(level=0)
points = reader.query(level=range(0, 3))
```

Note that `Bounds` can be 2D as well; in that case all Z values are considered:

```python
query_bounds = Bounds(
    mins=reader.header.mins[:2],
    maxs=(reader.header.mins + sizes / 2)[:2],
)
```

Saving the result is ordinary laspy from there:

```python
import laspy

new_header = laspy.LasHeader(
    version=reader.header.version,
    point_format=reader.header.point_format,
)
new_header.offsets = reader.header.offsets
new_header.scales = reader.header.scales

with laspy.open("subset.laz", mode="w", header=new_header) as f:
    f.write_points(points)
```

Because the returned points are a scale-aware record backed by NumPy, everything you would normally do with laspy or NumPy still applies — filtering by classification, computing statistics, or feeding the array into a model like the ones in the [PointNet++ article](/posts/semantic-segmentation-pointnet-plus-plus).

## Serving COPC over HTTP

There is no COPC server. You upload the file to object storage and you are done — but two details have to be right or clients will silently fail.

**Range requests must be supported.** S3, Google Cloud Storage, Azure Blob Storage, and Cloudflare R2 all honour the `Range` header by default. Some CDNs and reverse proxies do not, so verify:

```bash
curl -I -H "Range: bytes=0-374" https://example.com/data.copc.laz
```

A correct response returns `206 Partial Content` with a `Content-Range` header. A `200` with the whole file means range requests are not working, and every client will end up downloading everything.

**CORS must allow the `Range` header and expose the response headers**, otherwise browser-based viewers are blocked even though `curl` works fine. A minimal S3 CORS configuration:

```json
[
  {
    "AllowedOrigins": ["*"],
    "AllowedMethods": ["GET", "HEAD"],
    "AllowedHeaders": ["Range"],
    "ExposeHeaders": ["Content-Range", "Content-Length", "Accept-Ranges"]
  }
]
```

Tighten `AllowedOrigins` to your actual domain in production.

For viewing in the browser, [viewer.copc.io](https://viewer.copc.io) accepts a URL directly and is the fastest way to confirm a deployment works. For embedding in your own application, [Potree](https://github.com/potree/potree) and [loaders.gl](https://loaders.gl/) (which powers deck.gl) both read COPC.

## Entwine and EPT: the Predecessor

COPC did not appear out of nowhere. Its data organization is modeled directly on **EPT** (Entwine Point Tile), the format produced by [Entwine](https://github.com/connormanning/entwine), written by Connor Manning — also of Hobu. The `VoxelKey` structure in the COPC spec is explicitly the same octree addressing scheme EPT uses.

Entwine indexes anything PDAL can read and is built for datasets of hundreds of billions of points:

```bash
conda create --yes --name entwine --channel conda-forge entwine
conda activate entwine

entwine build \
    -i https://data.entwine.io/red-rocks.laz \
    -o ~/entwine/red-rocks
```

The crucial difference is what comes out. Entwine writes a **directory**: an `ept.json` metadata file plus a tree of thousands — sometimes millions — of small node files. COPC packs that same octree into a **single file**, as variably-chunked LAZ data with the hierarchy in a VLR.

The specification lists the concrete differences:

- COPC has no `ept.json`; that information lives in the LAS header and VLRs instead.
- COPC provides no equivalent of `ept-sources.json`, so per-source file metadata is not currently supported.
- COPC only supports the LAZ point format — EPT also allows binary point arrangements.
- COPC chunks store only point data as LAZ; EPT's LAZ nodes are complete LAZ files, each with its own header and VLRs.

### Which One Should You Use?

| | **COPC** | **EPT / Entwine** |
| --- | --- | --- |
| Output | One `.copc.laz` file | Directory of many files |
| Metadata | LAS header + VLRs | `ept.json` |
| Point storage | LAZ only | LAZ or binary |
| Source metadata | Not supported | `ept-sources.json` |
| Distribution | Copy one object | Sync a whole tree |
| CDN caching | Simple | Many objects to cache |
| Versioning | One file to version | Directory diffs |

For new work, **use COPC**. A single file is easier to move, cache, version, and reason about, and the format has broad tool support. Adoption backs this up: national mapping agencies in Canada, Sweden, New Zealand, and France's IGN have adopted COPC as a primary dissemination format, and QGIS uses it as its render-ready working format for point clouds. Tooling exists across PDAL, laspy, Potree, LAStools, OpenDroneMap, loaders.gl, FME, and Agisoft.

Reach for EPT when you are already invested in it — existing pipelines, existing datasets, or a need for the per-source metadata that COPC does not yet offer. Migration is straightforward when you decide to move, since PDAL reads EPT natively:

```json
[
  {
    "type": "readers.ept",
    "filename": "https://na.entwine.io/red-rocks/ept.json"
  },
  {
    "type": "writers.copc",
    "filename": "red-rocks.copc.laz"
  }
]
```

## Limitations

COPC is not a universal answer, and it is worth being clear about where it stops:

- **Tool support is still uneven outside geospatial.** The GIS, cloud, infrastructure, and heritage sectors have adopted it, but conventional BIM pipelines — Revit, Navisworks — do not read COPC.
- **PDRF 6/7/8 only.** Older LAS point formats have to be converted, and workflows depending on PDRF-specific quirks need checking.
- **No per-source metadata.** If you need to trace points back to the original flight line or scan file, EPT's `ept-sources.json` still has an advantage.
- **Building the index costs time and memory.** It is a one-time cost, but on large datasets it is a real one — which is precisely why Untwine exists.
- **It is an index, not a compression improvement.** A COPC file is roughly the size of the equivalent LAZ file. The win is in access patterns, not storage.

## Conclusion

COPC is a small idea executed well. By reorganizing a LAZ 1.4 file into a clustered octree and describing that octree in a 160-byte VLR at a fixed offset, it turns any static file server into a point cloud streaming service — while remaining a valid LAZ file that older tools can still read end to end.

If you already work with [LAS and LAZ](/posts/las-laz-format) and process data with [PDAL](/posts/pdal-point-clouds), adopting it is a genuinely small step: change one writer, upload the result, check your CORS configuration. What you get back is the ability to hand someone a URL to a 50 GB dataset and have them open the part they care about in under a second.
