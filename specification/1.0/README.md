Version 1.0

The ShapeDiver Transfer Format (sdTF) is an API-neutral exchange and storage format for trees of data items as used by parametric 3D modeling software like [Grasshopper®](https://www.grasshopper3d.com/).

Last Updated: March 10, 2021

Editors

  * Alexander Schiftner, ShapeDiver

Copyright (C) 2020-2021 ShapeDiver GmbH. All Rights Reserved.

# Contents

* [Introduction](#introduction)
  * [Motivation](#motivation)
  * [Basics](#basics)
  * [Design goals](#design-goals)
  * [Versioning](#versioning)
  * [File extensions and MIME types](#file-extensions-and-mime-types)
* [Concepts](#concepts)
  * [Asset](#asset)
  * [Chunks](#chunks)
  * [Nodes](#nodes)
  * [Data items](#data-items)
  * [Accessors](#accessors)
  * [Bufferviews](#bufferviews)
  * [Buffers](#buffers)
  * [Attributes](#attributes)
  * [Type hints](#type-hints)
* [Binary sdTF file format specification](#binary-sdtf-file-format-specification)
* [Properties reference](#properties-reference)


# Introduction

sdTF makes it possible to store and exchange trees of data items as used by parametric 3D modelling software in an efficient and extensible way, independent of software vendors. 

## Motivation

Chaining applications based on parametric 3D modelling requires an open, efficient, and easily extensible data format for exchange of complex data structures. sdTF aims to close this gap. 

## Basics

sdTF stores data items like primitives, 3D objects, images, PDF documents, as well as metadata about them: 

  * the tree structure used to organize data items
  * optional attributes attached to data items

The metadata is stored in JSON format, while data items may either be embedded into the metadata (typically used for primitive data items), or stored in binary buffers which are referenced from the metadata (3D objects, images, documents, arrays of numbers, other blobs). Metadata and binary buffers may either be kept in separate files, or merged into a single [_binary sdTF_](#binary-sdtf-file-format-specification). 

## Design goals

sdTF has been designed with the following goals in mind: 

  * _Lazy loading, separate fetching of metadata and data items_: sdTF allows for the metadata to be fetched and inspected without loading binary data. Fetching of individual data items is possible using HTTP range requests, without downloading a complete asset. 
  * _Independence from specific runtimes_: sdTF does not depend on closed source libraries.
  * _Extensibility_: The sdTF asset format specification is openly available and allows for extensions in many ways. As an example support for further formats for serialisation of data items can easily be added. The specification itself allows for extensions while keeping backwards compatibility. 

## Versioning

Any updates made to sdTF in a minor version will be backwards and forwards compatible. Backwards compatibility will ensure that any client implementation that supports loading a sdTF 1.x asset will also be able to load a sdTF 1.0 asset. Forwards compatibility will allow a client implementation that only supports sdTF 1.0 to load sdTF 1.x assets while gracefully ignoring any new features it does not understand.

A minor version update can introduce new features but will not change any previously existing behavior. Existing functionality can be deprecated in a minor version update, but it will not be removed.

Major version updates are not expected to be compatible with previous versions.

## File extensions and MIME types

  * `*.sdtf` for _binary sdTF_, MIME type `model/vnd.sdtf`
  * `*.jsdtf` for JSON sdTF (metadata only), MIME type `model/vnd.sdtf+json`
  * `*.bin` for binary buffers referenced from metadata, MIME type `application/octet-stream`

# Concepts

sdTF metadata is stored in JSON format. This section describes the concepts used for the properties representing the metadata in JSON. A flat hierarchy is used, using numeric indices for refering between child items of top level attributes. 

## Asset

This property must exist in any sdTF. It contains information about versioning, and optional information about the author and copyright. 

Example (minimal sdTF): 

```
{
  "asset": {
    "generator": "ShapeDiverGltfV2Writer",
    "copyright": "2021 (c) ShapeDiver GmbH",
    "version": "1.0"
  }
}
```

## Chunks

Chunks use the same schema as nodes, the only difference being that they provide the entry points to the sdTF. 
Typically every chunk corresponds to a separate tree structure. 

## Nodes

## Data items

## Accessors

## Bufferviews

## Buffers

## Attributes

## Type hints

# Binary sdTF file format specification


# Properties reference




