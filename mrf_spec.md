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
The file consists of a header, a keyframe offsets table, and a set of data sections. The sections do not have explicit identifiers or length fields. Their locations are determined by offsets stored in the header and the keyframe offsets table.

The sections following the keyframe offsets table ([Texture Path](#texture-path), [Face Data](#face-data), [Mapping Data](#mapping-data), and all [Keyframes](#keyframe)) may appear in any order within the file. Arbitrary data may exist between them. The game parser imposes no constraints on their relative placement, as long as all stored offsets are correct.

Although the game parser does not require section alignment, all known official `.mrf` files include zero-padding to align each logical section to a 16-byte boundary. This padding likely originates from the internal exporter used in development. Tools that produce `.mrf` files may omit padding without affecting game parser behavior.

The layout used in all known official files is:

- Header
- Keyframe Offsets Table
- Texture Path
- Face Data
- Mapping Data
- Keyframe 0 (first)
- ...
- Keyframe N (last)

# Sections Description

## Header
The header contains the magic ID, 3D model metadata, and offsets to fixed sections of the file.

The original game parser treats only the first `80` bytes of the file as the binary header. It copies this portion directly into memory. The remaining fields are read by offset rather than as part of a formalized structure.

#### Section structure
| Type  | Description |
|------|-------|
| **byte[4]** | Magic string `Morf`, represented as ASCII bytes: `4D 6F 72 66`. The game parser reads this field but does **not validate** it. Any 4-byte sequence is accepted. |
| **uint32** | Number of keyframes (used as `nFrames`). |
| **uint32** | Number of vertices (used as `nVerts`). |
| **uint32** | Number of face indices (used as `nIndices`). |
| **float** | ``frameDuration``. Time between keyframes in seconds (inverse keyframe rate). Must be greater than `0` for correct playback. A value of `0` prevents rendering, while a negative value displays only the last keyframe. |
| **vector3** | Pivot point. Parsed and stored, but not referenced by any internal function after initialization. |
| **float** | Bounds radius. Parsed and stored, but not referenced by any internal function after initialization. |
| **float** | Initial value of the playback time counter (elapsed time), in seconds. At runtime this value is incremented each frame and clamped to ``(nFrames - 1) × frameDuration``, after which the animation freezes on the last keyframe. Negative values delay the start of playback, positive values start from an offset. |
| **uint32** | Debug flag. Typically ``0``. In debug builds, a non-zero value triggers an assertion that guards against double-initialization of the same morph slot. Under normal conditions this assertion should never fire. No visible effect has been observed in retail versions. |
| **uint32[6]** | Parsed and stored, but not referenced by any internal function. Typically zeros. |
| **uint32**  | Offset of [Texture Path](#texture-path) relative to the beginning of the file. |
| **uint32**  | Offset of [Face Data](#face-data) relative to the beginning of the file. |
| **uint32**  | Offset of [Mapping Data](#mapping-data) relative to the beginning of the file. |

## Keyframe Offsets Table

Immediately after the header is a table of offsets, one per keyframe. Each entry is a `uint32` pointing to the start of a keyframe block relative to the beginning of the file. The number of keyframes is in the [Header](#header).

#### Section structure
| Type         | Description |
|--------------|-------------|
| **uint32[nFrames]** | Array of [Keyframe](#keyframe) offsets for keyframes `0` through `nFrames - 1` |
| **byte[]** | Padding to align to 16-byte boundary (if necessary) |

## Texture Path
The string is parsed from the beginning of the Section up to the first dot character (`.`). Any data following the dot, including zeros or arbitrary content, is ignored. As a result, the file extension is not required for texture lookup.
#### Section structure
| Type  | Description |
|------|-------|
| **str** | Texture path  |
| **byte[]** | Padding to align to the next 16-byte boundary (if necessary) |

## Face Data
Index buffer: array of `uint16`, interpreted as a triangle list. The number of indices is in the [Header](#header).
#### Section structure
| Type  | Description |
|------|-------|
| **triangle[nIndices / 3]** | Face (3 vertex IDs). Starting from face `0` and ending with face `nIndices / 3 - 1`  |
| **byte[]** | Padding to align to the next 16-byte boundary (if necessary) |

## Mapping Data
`U` and `V` are stored for each vertex as a `vector2`.
The number of vertices is in the [Header](#header). 

> **Note**: The V coordinate is flipped (`v = 1 - v`), following the DirectX UV convention.

#### Section structure
| Type  | Description |
|------|-------|
| **vector2[nVerts]** | Vertex UV coordinates. Starting from vertex `0` and ending with vertex `nVerts - 1` |
| **byte[]** | Padding to align to the next 16-byte boundary (if necessary) |

## Keyframe 
Each keyframe is stored as a separate Section and contains a complete snapshot of all vertex positions and normals.

The total number of keyframes is specified in the [Header](#header).
#### Section structure
| Type  | Description |
|------|-------|
| **(vector3, vector3)[nVerts]** | Vertex position and vertex normal. Each vertex stores a position followed by its normal, in an interleaved layout. Starting from vertex `0` and ending with vertex `nVerts - 1` |
| **byte[]** | Padding to align to the next 16-byte boundary (if necessary) |