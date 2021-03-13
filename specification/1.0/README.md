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

<p align="center">
<img src="assets/sdTF_spec_example.png" />
</p>

You can download [the complete binary sdTF asset](assets/sdTF_spec_example.sdtf) and the [Grasshopper model that was used to create it (requires ShapeDiver plugin >= 2.0)](assets/sdTF_spec_example.ghx).

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
    "version": "1.0"
  }
}
```


# Binary sdTF file format specification


# Properties reference


---------------------------------------
<a name="reference-accessor"></a>
### accessor

Accessors reference individual objects inside bufferviews. What they are referencing depends on the type of bufferview. In some cases they reference a complete bufferview. 

**Properties**

|   |Type|Description|Required|
|---|----|-----------|--------|
|**bufferView**|`integer`|Id of the referenced bufferview.|:white_check_mark: Yes|
|**id**|`string`|Id of the referenced object inside the bufferview.|No|

Additional properties are allowed.

#### accessor.bufferView :white_check_mark:

Id of the referenced bufferview.

* **Type**: `integer`
* **Required**: Yes
* **Minimum**: ` >= 0`

#### accessor.id

Id of the referenced object inside the bufferview. Used to reference individual objects in files that contain multiple objects. 
The meaning of this id is specific to the content type of the bufferview (the file type).
May be omitted in case the complete bufferview shall be referenced, e.g. in case of image files. 

* **Type**: `string`
* **Required**: No



---------------------------------------
<a name="reference-buffer"></a>
### buffer

A buffer is used to reference binary data. 

**Properties**

|   |Type|Description|Required|
|---|----|-----------|--------|
|**byteLength**|`integer`|Length of the buffer in bytes.|:white_check_mark: Yes|
|**uri**|`string`|Uri to fetch buffer from.|No|

Additional properties are allowed.

#### buffer.byteLength :white_check_mark:

Length of the buffer in bytes.

* **Type**: `integer`
* **Required**: Yes
* **Minimum**: ` >= 0`

#### buffer.uri

Uri to fetch data from. Can be a data uri. Not set in case of the directly attached buffer used for _binary sdTF_.

* **Type**: `string`
* **Required**: No



---------------------------------------
<a name="reference-bufferview"></a>
### bufferview

A bufferview references a chunk of data in a buffer.

**Properties**

|   |Type|Description|Required|
|---|----|-----------|--------|
|**buffer**|`integer`|Index of the referenced buffer.|:white_check_mark: Yes|
|**byteLength**|`integer`|Length of the bufferView in bytes.|:white_check_mark: Yes|
|**byteOffset**|`integer`|Offset into the buffer in bytes.|:white_check_mark: Yes|
|**contentEncoding**|`string`|Content-Encoding which was used to compress the data referenced by the buffer view.|No|
|**contentType**|`string`|MIME type of data referenced by the buffer view.|:white_check_mark: Yes|
|**name**|`string`|Optional name of the buffer view.|No|

Additional properties are allowed.

#### bufferview.buffer :white_check_mark:

Index of the referenced buffer (0-based).

* **Type**: `integer`
* **Required**: Yes
* **Minimum**: ` >= 0`

#### bufferview.byteLength :white_check_mark:

Length of the bufferView in bytes.

* **Type**: `integer`
* **Required**: Yes
* **Minimum**: ` >= 0`

#### bufferview.byteOffset :white_check_mark:

Offset into the buffer in bytes.

* **Type**: `integer`
* **Required**: Yes
* **Minimum**: ` >= 0`

#### bufferview.contentEncoding

Content-Encoding which was used to compress the data referenced by the buffer view. 
See [https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Encoding](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Content-Encoding).

* **Type**: `string`
* **Required**: No

#### bufferview.contentType :white_check_mark:

MIME type of data referenced by the buffer view.
See [https://www.iana.org/assignments/media-types/media-types.xhtml](https://www.iana.org/assignments/media-types/media-types.xhtml)
and [https://tools.ietf.org/html/rfc6838](https://tools.ietf.org/html/rfc6838)

* **Type**: `string`
* **Required**: Yes

#### bufferview.name

Optional name of the buffer view

* **Type**: `string`
* **Required**: No



---------------------------------------
<a name="reference-chunk"></a>
### chunk

Same as [node](#reference-node). 



---------------------------------------
<a name="reference-fileinfo"></a>
### fileinfo

Metadata about the sdTF asset. 

**Properties**

|   |Type|Description|Required|
|---|----|-----------|--------|
|**copyright**|`string`|Copyright mark.|No|
|**generator**|`string`|Hint to software package that generated the sdTF asset.|No|
|**version**|`string`|The sdTF version used by this asset.|:white_check_mark: Yes|

Additional properties are allowed.



---------------------------------------
<a name="reference-item"></a>
### item

Data items serve as the leaves of trees defined by nodes. The actual data may be embedded directly, or a reference to an accessor. Data items can have optional attributes. 

**Properties**

|   |Type|Description|Required|
|---|----|-----------|--------|
|**accessor**|`integer`|Index to referenced accessor.|No|
|**attributes**|`integer`|Index to referenced attributes.|No|
|**typeHint**|`integer`|Index to referenced typehint.|No|
|**value**|`any`|Embedded value.|No|

Additional properties are allowed.

#### item.accessor

Index to referenced accessor. Both an embedded value and an accessor may be specified, in which case the embedded value serves as preview.

* **Type**: `integer`
* **Required**: Yes
* **Minimum**: ` >= 0`

#### item.attributes

Index to referenced attributes.

* **Type**: `integer`
* **Required**: No

#### item.typeHint

Index to referenced typehint. **Should** be specified in case the type of the item can not be deduced from the representation of it's embedded value, or from the content type of the bufferview referenced by the accessor. 

* **Type**: `integer`
* **Required**: No

#### item.value

Embedded value.

* **Type**: `any`
* **Required**: No



---------------------------------------
<a name="reference-node"></a>
### node

Trees in sdTF are made of nodes. Nodes can reference other nodes and/or data items. 

**Properties**

|   |Type|Description|Required|
|---|----|-----------|--------|
|**attributes**|`integer`|Index to referenced attributes.|No|
|**items**|`integer[]`|Array of indices of child items.|No|
|**name**|`string`|Optional name of node.|No|
|**nodes**|`integer[]`|Array of indices of child nodes.|No|
|**typeHint**|`integer`|Index to referenced typehint.|No|

Additional properties are allowed.

#### node.attributes

Index to referenced attributes.

* **Type**: `integer`
* **Required**: No

#### node.items

Array of indices of child items.

* **Type**: `integer[]`
* **Required**: No

#### node.name

Optional name of node.

* **Type**: `string`
* **Required**: No

#### node.nodes

Array of indices of child nodes.

* **Type**: `integer[]`
* **Required**: No

#### node.typeHint

Index to referenced typehint. **Should** be specified in case the type hint for all child nodes and items is the same. **Must not** be specified otherwise.

* **Type**: `integer`
* **Required**: No



---------------------------------------
<a name="reference-typehint"></a>
### typehint

Type hints are used to add information about the type of data items found below a specific node in the tree. 

**Properties**

|   |Type|Description|Required|
|---|----|-----------|--------|
|**name**|`string`|Name of the typehint.|:white_check_mark: Yes|

Additional properties are allowed.

#### typehint.name :white_check_mark:

Name of the typehint

* **Type**: `string`
* **Required**: Yes
* **Supported values** (extensible):
  * bitmap
  * boolean
  * byte
  * char
  * color
  * decimal
  * double
  * geometry.complex
  * guid
  * int16
  * int32
  * int64
  * rhino.arccurve
  * rhino.beziercurve
  * rhino.brep
  * rhino.curve
  * rhino.extrusion
  * rhino.linecurve
  * rhino.mesh
  * rhino.nurbscurve
  * rhino.nurbssurface
  * rhino.planesurface
  * rhino.point
  * rhino.polycurve
  * rhino.polylinecurve
  * sbyte
  * single
  * string
  * uint16
  * uint32
  * uint64

