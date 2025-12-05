# Lab 10 – Depth-Based Fog / Atmospheric Haze with Custom Nodes & HLSL  
## Module: 3D Game Development  

## Overview  

In this lab you will implement a **depth-based fog / atmospheric haze** effect as a **Post-Process Material** using a **Custom HLSL node** in Unreal Engine.  

The idea is simple but powerful:  

- Objects **farther from the camera** will gradually fade into a fog colour.  
- Objects **close to the camera** remain sharp and unaffected.  

You will use the **Scene Depth** buffer to measure how far each pixel is from the camera, then blend between the original scene colour and a fog colour based on that depth. This is a core real-time rendering technique used in almost every 3D game.  

## Background – Scene Depth and Fog  

When the scene is rendered, Unreal stores the **distance from the camera** for each pixel in a **depth buffer** (Scene Depth). In a perspective camera:  

- Small depth values = geometry near the camera.  
- Large depth values = geometry far away.  

Depth-based fog uses this information to compute a **fog factor** `f` between 0 and 1:  

- `f = 0` → no fog (use pure scene colour).  
- `f = 1` → full fog (use pure fog colour).  

A simple linear model is:  

```text
f = saturate( (depth - fogStart) / (fogEnd - fogStart) )
```  

A more realistic atmospheric model uses **exponential falloff**:  

```text
f = 1 - exp(-density * depth)
```  

In this lab you will implement a hybrid approach that:  

- normalises depth into a 0–1 range using start and end distances  
- shapes it with an exponential curve for a smooth haze  

## Learning Outcomes  

By completing this lab, you will be able to:  

- Sample the **Scene Depth** and **Scene Colour** in a Custom HLSL node.  
- Compute a depth-based fog factor using linear and exponential functions.  
- Blend between scene colour and fog colour based on depth.  
- Control fog start, end, density, and intensity through material parameters.  

## 1. Create a Post Process Volume  

1. Drag a **Post Process Volume** into your level.  
2. In the Details panel:  
   - Enable **Infinite Extent (Unbound)** → `true`.  
3. Under **Post Process Materials**, add an **Array Element**.  
   - Set the type to **Asset Reference**.  
4. You will assign the depth-fog material here later.  

## 2. Create the Depth Fog Material  

1. In the Content Browser, create a new **Material**:  

   ```text
   M_DepthFog
   ```  

2. Open the material and set:  
   - **Material Domain → Post Process**  

You will output the final colour to **Emissive Color**.  

## 3. Provide Screen UVs to the Custom Node  

You will work in screen-space UVs so you can sample both Scene Colour and Scene Depth consistently.  

1. Add a **ScreenPosition** node.  
2. In the ScreenPosition node, make sure it is set to output **ViewportUV** (if needed).  
3. This gives a `float2` UV in the range `[0, 1]` for the current pixel.  
4. You will connect this UV to the Custom node input `uv`.  

## 4. Add Fog Parameters  

Create the following **Parameters** to control your fog:  

| Parameter Name | Default | Description |
| :- | :- | :- |
| `FogColor` | (0.6, 0.7, 0.8) | Colour of the fog / haze |
| `FogStart` | `0.1` | Normalised depth at which fog begins |
| `FogEnd` | `1.0` | Normalised depth at which fog reaches full strength |
| `FogDensity` | `1.0` | Controls the exponential falloff |
| `FogIntensity` | `1.0` | Overall strength multiplier (0–1) |

Notes:  

- In a real project, depth is not always directly linear. For teaching, we will treat the depth returned by the lookup as approximately normalised and fit our FogStart and FogEnd values experimentally.  

## 5. Main Custom Node – Depth-Based Fog Logic  

You will now create a **Custom HLSL node** that:  

1. Receives screen UVs.  
2. Samples Scene Colour and Scene Depth at the same UV.  
3. Computes a fog factor based on depth.  
4. Blends scene colour with fog colour.  

### 5.1 Add the Custom Node  

1. Add a **Custom** node:  
   - Name: `DepthFog`  
   - Output Type: `Float3`  

2. Add the following **inputs** to the Custom node:  

| Input Name | Type | Connect From |
| :- | :- | :- |
| `uv` | Float2 | `ScreenPosition (ViewportUV)` output |
| `fogColor` | Float3 | `FogColor` Vector Parameter |
| `fogStart` | Float1 | `FogStart` Scalar Parameter |
| `fogEnd` | Float1 | `FogEnd` Scalar Parameter |
| `fogDensity` | Float1 | `FogDensity` Scalar Parameter |
| `fogIntensity` | Float1 | `FogIntensity` Scalar Parameter |

### 5.2 HLSL Code  

Paste the following **fully commented** HLSL into the Custom node’s Code field:  

```hlsl
// Inputs:
//   uv           : float2 screen-space UV in [0, 1]
//   fogColor     : float3 RGB colour of the fog
//   fogStart     : scalar depth at which fog begins (approx. 0..1)
//   fogEnd       : scalar depth at which fog is at full strength
//   fogDensity   : scalar controlling exponential fog falloff
//   fogIntensity : scalar multiplier (0 = off, 1 = full fog)

// Step 1: Sample the scene colour from PostProcessInput0.
// 14 = PostProcessInput0 in Unreal post-process materials.
float3 sceneColor = SceneTextureLookup(uv, 14, false).rgb;

// Step 2: Sample the scene depth using SceneDepthLookup.
// This returns depth for the current pixel.
// 'false' means we use the raw device depth (sufficient for this lab).
float depth = SceneDepthLookup(uv, false);

// Step 3: Normalise depth based on fogStart and fogEnd.
// We want a 0..1 fogFactorLinear where:
//   depth <= fogStart  → fogFactorLinear = 0 (no fog)
//   depth >= fogEnd    → fogFactorLinear = 1 (full fog)
float depthRange = max(fogEnd - fogStart, 1e-4);       // avoid divide-by-zero
float fogFactorLinear = saturate((depth - fogStart) / depthRange);

// Step 4: Apply an exponential falloff to make the fog feel more natural.
// This shapes the fog so it ramps in smoothly over distance.
// density controls how quickly it ramps up.
float fogFactorExp = 1.0 - exp(-fogDensity * fogFactorLinear);

// Step 5: Combine linear and exponential factors.
// Here we simply use the exponential factor, but this is a hook
// where you could blend between them if desired.
float fogFactor = fogFactorExp;

// Step 6: Apply overall intensity so that designers can dial the effect
// up or down without re-tuning all other parameters.
fogFactor = saturate(fogFactor * fogIntensity);

// Step 7: Blend between the original scene colour and the fog colour.
// When fogFactor = 0 → pure sceneColor.
// When fogFactor = 1 → pure fogColor.
float3 finalColor = lerp(sceneColor, fogColor, fogFactor);

// Step 8: Return the final fogged colour.
return finalColor;
```  

## 6. Connect to Emissive Color  

Connect the **DepthFog** Custom node output (Float3) to the material’s **Emissive Color** pin.  

This tells Unreal to use your fogged colour as the final post-process result.  

## 7. Assign the Material to the Post Process Volume  

1. Return to your level and select the **Post Process Volume**.  
2. Under **Post Process Materials**, assign:  

   ```text
   M_DepthFog
   ```  

3. Set **Blend Weight = 1.0**.  

Press **Play**. Objects farther from the camera should fade smoothly into the fog colour. Adjust your FogStart, FogEnd, and FogDensity values until the effect feels good in your scene.  

## 8. Student Tasks  

### Task 1 – Tune Fog for Different Environments  

Using a Material Instance of `M_DepthFog`, create presets for:  

- A **bright desert** (pale yellow or light sand-coloured fog).  
- A **dark forest** (deep green or blue-green fog).  
- A **stormy mountain** (grey-blue fog with strong density).  

For each environment, record:  

- `FogColor`  
- `FogStart`, `FogEnd`  
- `FogDensity`, `FogIntensity`  

and describe how they affect the atmosphere.  

### Task 2 – Near vs Distant Fog  

Experiment with fog that:  

- Starts very close to the camera (`FogStart` small) with low `FogDensity`.  
- Starts far away but ramps in aggressively with high `FogDensity`.  

Questions:  

- Which combination feels more **claustrophobic**?  
- Which feels more **open** or **epic**?  

### Task 3 – Height-Based Fog (Optional Extension)  

Extend the fog so that it also depends on **world height** (Y or Z):  

1. Use `PixelDepth` and/or a WorldPosition node in the graph to reconstruct an approximate world-space height.  
2. Pass a height factor into the Custom node.  
3. Modify the HLSL to reduce fog above a certain height (e.g. top of mountains) or increase fog closer to the ground (ground mist).  

This becomes a simplified version of **height fog**.  

## 9. Reflective Questions  

1. Why is depth information essential for creating realistic fog and atmospheric haze in a 3D scene?  
2. What is the visual difference between a purely linear fog factor and one shaped by an exponential function?  
3. How do the `FogStart` and `FogEnd` parameters affect gameplay and readability in your scene?  
4. How might you animate fog parameters over time to simulate changing weather or entering a dangerous area?  
5. How could depth-based fog be combined with lens distortion, vignette, or chromatic aberration to create a specific cinematic mood?  

