*Written by wiselen.  
RU version hosted on XGM [here](https://xgm.guru/p/wc3/morf).*

# Introduction

This documentation and [specification](mrf_spec.md) are based on reverse engineering of six known `.mrf` files from Warcraft III. All information presented here is derived from analysis, testing, and experimentation.

# Description

MRF is a three-dimensional model format that contains one vertex animation. These models were used by Blizzard for Arthas' cape in the classic cinematic model of the battle between Arthas and Illidan.

In the Reforged version correct render of the MRF is only possible in the classic version of the graphic (SD), while in the HD there are some troubles with culling.

An array of triangles (faces), a UV Mapping and a path to a image texture are stored in the model as static data.
The vertex data is divided into an array of keyframes. Each keyframe is represented as an array of coordinates and normals for each vertex. Keyframes replace each other at a given frequency, and the graphics engine linearly interpolates vertex positions and normals between adjacent keyframes. 

MRF only supports one static texture without blending. Transparent pixels are rendered as black. 

The MRF format also contains two main parameters that control animation playback:

- **Frame duration** — Defines the speed of the animation is seconds.

    - For example, if frame duration equals `0.033` seconds (typical value in Blizzard files), the animation plays at roughly `30` FPS.

    - If set to a negative value, the animation displays only the last keyframe.

    - If set to `0`, the model is not rendered.

- **Elapsed time** — Controls the starting point of the animation. This works, but is not used in Blizzard files.

    - Negative values delay the start of playback.

    - Positive values begin playback from a specified offset.

    - If the value exceeds the total animation time, the animation is frozen at the last keyframe.

See [specification](mrf_spec.md) for details.

# Import description

MRF files are linked to the MDX/MDL model through tracks of Event Objects (EVTS) with `MRF` and `MRD` identifiers.

The ID of the event object is embedded in the path to the MRF file. The hardcoded part of this path is `doodads\cinematic\arthasillidanfight\arthascape%s.mrf`, where the ID is substituted in place of `%s`. In essence, the ID is just an arbitrary string with a maximum length of `208` bytes.
Here are examples in the table.

| MRF (show)  | MRD (hide) | Path |
|------|-------|-------|
| MRFX**0000** | MRDX**0000** | *doodads\cinematic\arthasillidanfight\arthascape**0000**.mrf* |
| MRFX**12345** | MRDX**12345** | *doodads\cinematic\arthasillidanfight\arthascape**12345**.mrf* |
| MRFX**helloworld** | MRDX**helloworld** | *doodads\cinematic\arthasillidanfight\arthascape**helloworld**.mrf* |

MDL Example:

```
EventObject "MRFxY" {
    ObjectId A,
    EventTrack B {
        C,
    }
}
```
| Field  | Description |
|------|-------|
| `MRF`| The event object identifier. This tells the game that an MRF  should triggered |
| `x` | A single character that serves to make the event object name unique. It follows Warcraft's naming conventions|
| `Y` | An arbitrary string that represents the ID|
| `A` | The node index in the MDX/MDL model to which this event object is attached |
| `B` | The number of key frames in the event (usually `1`)|
| `C` | The specific frame where the MRF object becomes visible and starts rendering|

Warcraft ignores the pivot point of event object, as well as translation, rotation, and scaling tracks when rendering MRF models. Instead, the models are placed based on their internal coordinates without adjustments from the parent MDX model.

Since the link to the MRF  is part of the MDX/MDL, theoretically MRF can be played in any space where models can be played. But in fact, MRF *animation* is not supported everywhere.

There are four options to display the MRF model and play its animation:
- Output a `SPRITE` frame via Frames API using a camera from the MDX/MDL model. 
*If a world frame is used as the parent, then the global lighting model (DNC with default directional light) from the current map will be used as the light source.*

- Using model with MRF as a transition model with a native `PlayModelCinematic(MDX/MDL model)` function.  For example *PlayModelCinematic( "Doodads\\Cinematic\\ArthasIllidanFight\\ArthasIllidanFight.mdl" )*.

- Using a model as a unit portrait. MRF models can be displayed in the portrait view of units, utilizing the MDX camera. 

- Using a model as 3D main menu screen in pre-reforged patches.

As for the world space and the space of 3D campaign screens, MRF will also be rendered there, but only as a static object, that is, the mesh from zero frame of the MRF will be loaded.

## Rendering Issues with Frame API

When rendering MRF models through the **Frame API** as `SPRITE`, a particularly annoying issue may arise: the sprite can sometimes "sink" into world models. In other words, instead of rendering clearly above world geometry, it appears as though both world models and the sprite are being drawn in the same space. This creates a visual glitch where parts of the MRF model seem to be clipped or occluded by the environment.

This is **not a specific issue with MRF**, but rather a general problem with `SPRITE` frames in Warcraft III. The exact mechanism is unclear, but from experimental observations, it seems that the issue might be related to depth calculations between the sprite's camera and the game world. 
One possible explanation is that the distance from the sprite's camera to its vertices shold be less than the distance from the game camera to world objects. 

### **Known Workarounds:**

1. **Using the `ORIGIN_PORTRAIT_FRAME` as parent and make sure it is visible:**  
In most cases this helps.

2. **Hiding Terrain:**  
Using the native function `BlzShowTerrain(false)`. This does not fully solve the issue, but it prevents SPRITE frames from completely sinking under the landscape.

3. **Using [UjAPI](https://github.com/UnryzeC/UjAPI) for Pre-Reforged Patches:**  
The UjAPI dev claims to have patched this issue.

For MRF output via portrait or `PlayModelCinematic` native, the issue is not relevant.