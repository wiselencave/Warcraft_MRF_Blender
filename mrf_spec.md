*Unofficial Specification for the .mrf (Morf) File Format for Warcraft III*

<hr>

# Data types 

| Name  | Description |
|------|-------|
| **byte** | 1 byte |
| **uint16** | 2 byte unsigned integer (Little-Endian) |
| **uint32** | 4 byte unsigned integer (Little-Endian) |
| **float** | 4 byte floating point number (Little-Endian) |
| **str** | string (not null-terminated)|

### Derived data types
| Name  | Description |
|------|-------|
| **vector3** | 3 floats (X, Y, Z) |
| **vector2** | 2 floats (U, V) |
| **triangle** | 3 uint16 (vertex 0 ID, vertex 1 ID, vertex 2 ID) |

# Data Structure
The file consists of a header followed by multiple logical chunks, stored sequentially. Chunks do not have explicit identifiers or length fields; their locations are determined by offsets defined in the header.

> **Note:** These "chunks" are a logical abstraction used in this specification for clarity.
> In the actual implementation, the original game parser does not treat the file as containing separate chunks. It parses the header, then directly accesses other sections using hardcoded offsets and pointer arithmetic. There is no runtime detection or validation of individual sections.

Although the game parser does not require chunk alignment, all known official `.mrf` files include zero-padding to align each logical chunk to a 16-byte boundary. This padding likely originates from the internal exporter used in development, possibly to match memory alignment requirements on 32-bit platforms.

While not strictly required for correct parsing, preserving 16-byte alignment is recommended for compatibility with original data.

The logical structure is as follows:

- Header
- Keyframe Offsets Table
- Texture Path
- Face Data
- Mapping Data
- Keyframe 0 (first)
- ...
- Keyframe N (last)

# Chunks Description

## Header
Header contains the magic ID, 3D model metadata, and offsets to fixed sections of the file.

The original game parser treats only the first `80` bytes of the file as the binary header — this portion is copied directly into memory. The remaining fields are read manually using pointer arithmetic, rather than as part of a defined structure.

#### Chunk structure
| Type  | Description |
|------|-------|
| **byte[4]** | Magic string `Morf`, represented as ASCII bytes: `4D 6F 72 66`. Although this field is read by the parser, its value is **not validated**. In practice, any 4-byte sequence is accepted without issue |
| **uint32** | Number of keyframes (used as `nFrames`) |
| **uint32** | Number of vertices (used as `nVerts`) |
| **uint32** | Number of face indices (used as `nIndices`) |
| **float** | ``frameDuration``. Time between frames in seconds. Inverted frame rate. Must be greater than `0` for correct playback. A value of `0` results in the model not being rendered, while a negative value causes the animation to display only the last keyframe|
| **vector3** | Pivot point. Read and stored, but has no effect in-game |
| **float** | Bounds radius. Read and stored, but has no effect in-game |
| **float** | Elapsed Time. Initial playback time in seconds. Defines when the animation starts. Negative values delay playback; positive values begin playback from a specific offset. Values exceeding `(nFrames - 1) × frameDuration` causes the animation to display only the last keyframe
| **uint32** | Debug flag. Reserved for internal development use. Should be ``0``. Non-zero values trigger assertions and additional checks that verify texture handle state. Has no effect in retail versions
| **uint32[6]** | Ignored. Can contain any arbitrary data, typically zeros |
| **uint32**  | Offset of [Texture Path](#texture-path) relative to the beginning of the file |
| **uint32**  | Offset of [Face Data](#face-data) relative to the beginning of the file |
| **uint32**  | Offset of [Mapping Data](#mapping-data) relative to the beginning of the file |

## Keyframe Offsets Table

Immediately following the Header is a table of offsets, one per keyframe. Each entry is a `uint32` pointing to the start of a keyframe block, relative to the beginning of the file. The number of keyframes is in the [Header](#header).

#### Chunk structure
| Type         | Description |
|--------------|-------------|
| **uint32[nFrames]** | Array of [Keyframe](#keyframe) offsets. Starting from offset of keyframe `0` and ending with offset of keyframe `nFrames - 1` |
| **byte[]**    | Padding to align to 16-byte boundary (if necessary) |

## Texture Path
The string is parsed from the beginning of the chunk up to the first dot character (`.`). Any data following the dot, including zeros or arbitrary content, is ignored. As a result, the file extension is not required for texture lookup.
#### Chunk structure
| Type  | Description |
|------|-------|
| **str** | Texture path  |
| **byte[]** | Padding to align to the next 16-byte boundary (if necessary) |

## Face Data
Index buffer: array of `uint16`, interpreted as triangle list. The number of indices is in the [Header](#header).
#### Chunk structure
| Type  | Description |
|------|-------|
| **triangle[nIndices / 3]** | Face (3 vertex IDs). Starting from face `0` and ending with face `nIndices / 3 - 1`  |
| **byte[]** | Padding to align to the next 16-byte boundary (if necessary) |

## Mapping Data
`U` and `V` are stored for each vertex. We can represent this as `vector2`. 
The number of vertices is in the [Header](#header). 

> **Note**: The V coordinate is flipped (`v = 1 - v`), following the DirectX UV convention.

#### Chunk structure
| Type  | Description |
|------|-------|
| **vector2[nVerts]** | Vertex UV coordinates. Starting from vertex `0` and ending with vertex `nVerts - 1` |
| **byte[]** | Padding to align to the next 16-byte boundary (if necessary) |

## Keyframe 
Each keyframe is stored as a separate chunk and contains a complete snapshot of the state (position and normal) of all vertices in the mesh.

The total number of keyframes is specified in the [Header](#header).
#### Chunk structure
| Type  | Description |
|------|-------|
| **(vector3, vector3)[nVerts]** | Vertex position and vertex normal. Each vertex stores a position followed by its normal, in an interleaved layout. Starting from vertex `0` and ending with vertex `nVerts - 1` |
| **byte[]** | Padding to align to the next 16-byte boundary (if necessary) |