# sdTF
Specification of the Structured Data Transfer Format

sdTF (Structured Data Transfer Format) is a specification for the efficient transfer and loading of trees of data items as used by parametric 3D modeling software like [Grasshopper®](https://www.grasshopper3d.com/). 
sdTF borrows concepts from [glTF](https://github.com/KhronosGroup/glTF) and encodes data items, the tree structure used to organize these items, and optional attributes attached at different levels of the tree. 
Storage of the tree structure and attributes is kept separate from the serialized data items, allowing for efficient exploration and partial fetching of data stored in sdTF assets. 
Data items may be stored as primitives (numbers, booleans, strings), JSON encoded, or binary in arbitrary file formats. Binary file buffers may be kept separate or bundled together with the tree structure into a single binary asset.  
sdTF is extensible at various levels, e.g. by adding support for further file formats or primitive data types. 

# Specification

Please provide spec feedback by submitting [issues](https://github.com/shapediver/sdTF/issues). Ask general questions about sdTF in the [ShapeDiver forum](https://forum.shapediver.com/). 

  * [sdTF specification 1.0](specification/1.0/README.md)

# Reference implementations

  * [SDK for JavaScript / TypeScript / node.js](https://www.npmjs.com/package/@shapediver/sdk.sdtf-v1)
  * SDK for .NET (available on request, will be published in Q4 2022)
