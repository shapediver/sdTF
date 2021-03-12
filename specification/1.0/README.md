Version 1.0

The ShapeDiver Transfer Format (sdTF) is an API-neutral exchange and storage format for trees of data items as used by parametric 3D modeling software like [Grasshopper®](https://www.grasshopper3d.com/).

Last Updated: March 10, 2021

Editors

  * Alexander Schiftner, [ShapeDiver](https://www.shapediver.com)

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
* [A complete example](#a-complete-example)
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
    "generator": "ShapeDiverSdtfWriter",
    "copyright": "Author of sdTF asset",
    "version": "1.0"
  }
}
```

## Nodes

Trees in sdTF are made of nodes. Nodes can reference other nodes and/or data items. They can have an optional name, optional attributes, and an optional type hint. 
Due to the flat hierarchy it's theoretically possible to create circular structures of nodes, but that's discouraged. 

Examples: 

### Node referencing data items

```
{
  "name": "[0,0]",
  "items": [3, 4, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14],
  "typeHint": 2
}
```

### Node referencing other nodes

```
{
  "name": "57e60008-ee18-4864-8711-1cbc8adfc821",
  "nodes": [0, 1],
  "typeHint": 0,
  "attributes": 1
}
```

## Chunks

Chunks use the same schema as nodes, the only difference being that they provide the entry points to the sdTF. 
Typically every chunk corresponds to a separate tree structure. 

Example: In case we are storing several Grasshopper data trees in a sdTF, each data tree becomes a chunk. 
The first level of nodes below a chunk corresponds to the branches of the tree. Each such node references a list of data items.

## Data items

Data items serve as the leaves of trees defined by nodes. The actual data may be embedded directly, or a reference to an accessor. Data items can have optional attributes.

Examples: 

### Item referencing an accessor

```
{
  "accessor": 0,
  "typeHint": 0
}
```

### Item embedding a value

```
{
  "value": 42.0,
  "typeHint": 2
}
```


## Accessors

Accessors reference individual objects inside bufferviews. What they are referencing depends on the type of bufferview. In some cases they reference a complete bufferview.

Examples: 

### Geometric object in a 3dm file

```
{
  "bufferView": 0,
  "id": "64b723b9-2712-4f4e-a7ef-9c34cbf68289"
}
```

### Image

The accessor references a complete bufferview (in this case an image file).

```
{
  "bufferView": 0
}
```

## Bufferviews

A bufferview is a pointer to a portion of a buffer, it references a file or simply some binary data (e.g. a large list of numbers, or a large string). 
The `byteOffset` and `byteLength` define the location of the bufferview inside the buffer. 

Examples: 

### Bufferview for a 3dm file

```
{
  "buffer": 0,
  "byteOffset": 0,
  "byteLength": 5164,
  "contentType": "model/vnd.3dm",
  "contentEncoding": "gzip"
}
```

### Bufferview for an image file

```
{
  "buffer": 0,
  "byteOffset": 0,
  "byteLength": 2073,
  "contentType": "image/png"
}
```

## Buffers

Buffers are used to story binary data, imagine them as a concatenation of individual files (in fact: bufferviews). A single sdTF can reference multiple buffers. The actual data of a buffer can reside

  * in a external binary file referenced by an uri, 
  * embedded into the JSON metadata by means of a data uri, or
  * appended directly to the JSON metadata in case of a [_binary sdTF_](#binary-sdtf-file-format-specification).

Examples:

### External buffer

Uris to external buffers are resolved relative, absolute uris can be used but are discouraged.

```
{
  "byteLength": 1234,
  "uri": "buffer.bin"
}
```

### Attached buffer (binary sdTF)

In case of binary sdTF the first buffer in the list refers to the directly attached one, hence it only uses the `byteLength` property.

```
{
  "byteLength": 2358119
}
```

## Attributes

Attributes are stored as dictionaries. They can be referenced by nodes, chunks, and data items. Attribute values can be directly embedded, or reference accessors. 

Example: 

```
{
  "Name": {
    "value": "Mesh sphere",
    "typeHint": 3
  },
  "Color": {
    "value": "126, 156, 255",
    "typeHint": 4
  },
  "Layer": {
    "value": "Some layer",
    "typeHint": 3
  },
  "Preview": {
    "accessor": 3,
    "typeHint": 1
  }
}
```

## Type hints

Type hints are used to add information about the type of data items found below a specific node in the tree. They are also used for data items, allowing to know 
about the type without reading binary data. 

Example: 

```
{
  "typeHints": [{
			"name": "rhino.mesh"
		}, {
			"name": "image"
		}, {
			"name": "double"
		}, {
			"name": "string"
		}, {
			"name": "color"
		}, {
			"name": "guid"
		}]
}
```
# A complete example

The following example shows the metadata of an sdTF asset containing 3 trees: 

  * a tree called _Mesh_ containing two branches with one mesh each (one of the meshes has attributes attached)
  * a tree called _Images_ containing a single branch with a single image
  * a tree called _Numbers_ containing two branches which are lists of numbers

```
{
	"attributes": [{
			"Name": {
				"value": "Mesh sphere",
				"typeHint": 3
			},
			"Color": {
				"value": "126, 156, 255",
				"typeHint": 4
			},
			"Layer": {
				"value": "Some layer",
				"typeHint": 3
			},
			"Preview": {
				"accessor": 3,
				"typeHint": 1
			}
		}, {
			"Id": {
				"value": "57e60008-ee18-4864-8711-1cbc8adfc821",
				"typeHint": 5
			},
			"Name": {
				"value": "Mesh",
				"typeHint": 3
			},
			"Type": {
				"value": "Mesh",
				"typeHint": 3
			}
		}, {
			"Id": {
				"value": "c641e0f8-ecfd-4607-936f-01ed06ac7dbd",
				"typeHint": 5
			},
			"Name": {
				"value": "Images",
				"typeHint": 3
			},
			"Type": {
				"value": "GrasshopperBitmap",
				"typeHint": 3
			}
		}, {
			"Id": {
				"value": "fc5cedb5-42c0-4238-ae01-9cf94d194130",
				"typeHint": 5
			},
			"Name": {
				"value": "Numbers",
				"typeHint": 3
			},
			"Type": {
				"value": "Number",
				"typeHint": 3
			}
		}
	],
	"version": 1,
	"typeHints": [{
			"name": "rhino.mesh"
		}, {
			"name": "image"
		}, {
			"name": "double"
		}, {
			"name": "string"
		}, {
			"name": "color"
		}, {
			"name": "guid"
		}
	],
	"chunks": [{
			"name": "57e60008-ee18-4864-8711-1cbc8adfc821",
			"nodes": [0, 1],
			"typeHint": 0,
			"attributes": 1
		}, {
			"name": "c641e0f8-ecfd-4607-936f-01ed06ac7dbd",
			"nodes": [2],
			"typeHint": 1,
			"attributes": 2
		}, {
			"name": "fc5cedb5-42c0-4238-ae01-9cf94d194130",
			"nodes": [3, 4],
			"typeHint": 2,
			"attributes": 3
		}
	],
	"nodes": [{
			"name": "[0,0]",
			"items": [0],
			"typeHint": 0
		}, {
			"name": "[0,1]",
			"items": [1],
			"typeHint": 0
		}, {
			"name": "[0]",
			"items": [2],
			"typeHint": 1
		}, {
			"name": "[0,0]",
			"items": [3, 4, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14],
			"typeHint": 2
		}, {
			"name": "[0,1]",
			"items": [4, 5, 6, 7, 15, 16, 9, 17, 18, 19, 20],
			"typeHint": 2
		}
	],
	"items": [{
			"accessor": 0,
			"typeHint": 0,
			"attributes": 0
		}, {
			"accessor": 1,
			"typeHint": 0
		}, {
			"accessor": 2,
			"typeHint": 1
		}, {
			"value": 0.0,
			"typeHint": 2
		}, {
			"value": 1.0,
			"typeHint": 2
		}, {
			"value": 2.0,
			"typeHint": 2
		}, {
			"value": 3.0,
			"typeHint": 2
		}, {
			"value": 5.0,
			"typeHint": 2
		}, {
			"value": 8.0,
			"typeHint": 2
		}, {
			"value": 13.0,
			"typeHint": 2
		}, {
			"value": 21.0,
			"typeHint": 2
		}, {
			"value": 34.0,
			"typeHint": 2
		}, {
			"value": 55.0,
			"typeHint": 2
		}, {
			"value": 89.0,
			"typeHint": 2
		}, {
			"value": 144.0,
			"typeHint": 2
		}, {
			"value": 7.0,
			"typeHint": 2
		}, {
			"value": 11.0,
			"typeHint": 2
		}, {
			"value": 17.0,
			"typeHint": 2
		}, {
			"value": 19.0,
			"typeHint": 2
		}, {
			"value": 23.0,
			"typeHint": 2
		}, {
			"value": 27.0,
			"typeHint": 2
		}
	],
	"accessors": [{
			"bufferView": 0,
			"id": "ae2b6cef-ce2c-4a44-8fc7-e0eb23588a89"
		}, {
			"bufferView": 0,
			"id": "01b482a5-115a-4327-b557-79e3f40ad397"
		}, {
			"bufferView": 1
		}, {
			"bufferView": 2
		}
	],
	"bufferViews": [{
			"buffer": 0,
			"byteOffset": 0,
			"byteLength": 12477,
			"contentType": "model/vnd.3dm",
			"contentEncoding": "gzip"
		}, {
			"buffer": 0,
			"byteOffset": 12480,
			"byteLength": 2172131,
			"contentType": "image/png"
		}, {
			"buffer": 0,
			"byteOffset": 2184612,
			"byteLength": 173507,
			"contentType": "image/png"
		}
	],
	"buffers": [{
			"byteLength": 2358119
		}
	],
	"asset": {
		"generator": "ShapeDiverSdtfWriter",
		"version": 1
	}
}
```


# Binary sdTF file format specification


# Properties reference




