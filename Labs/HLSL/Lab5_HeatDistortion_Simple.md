
# Lab 5 – Heat Distortion / Mirage (Simple HLSL UV Warping)  
## Module: 3D Game Development  

## Overview

In this lab you will create a lightweight **heat distortion / mirage** post-process effect using:

- one **Custom HLSL node** that outputs **distorted UVs**, and  
- a **SceneTexture node** to sample the scene at those UVs.

This effect simulates shimmering hot air by bending UVs over time.



## Learning Outcomes

By completing this lab, you will be able to:

- Use **Post-Process Materials** in Unreal.  
- Use Custom HLSL nodes to **distort UVs**.  
- Animate distortion using **time**, **scale**, **speed**, and **strength**.  
- Understand how **screen-space warping** creates heat-haze effects.  



## 1. Create a Post Process Volume

1. Drag a **Post Process Volume** into the level.  
2. Enable **Infinite Extent (Unbound)** → `true`.  
3. Under **Post Process Materials**, add an **Array Element** and set it to **Asset Reference**.  
4. You’ll assign your material here later.



## 2. Create the Material

1. Create a new Material:

   ```text
   M_HeatDistortion_Simple
   ```

2. Open it and set:
   - **Material Domain → Post Process**  
3. Only **Emissive Color** will be used.



## 3. Add a Helper Node for Screen UVs

We need accurate 0–1 screen UVs. Unreal provides a helper function for this.

1. Add a **Custom** node.  
   - **Name:** `GetScreenUVs`  
   - **Output Type:** `Float2`
2. Paste this code:

   ```hlsl
   return GetSceneTextureUV(Parameters);
   ```

3. Rename the output pin description to **ScreenUV** (optional).

This node now outputs `ScreenUV`.



## 4. Add Parameters + Time

Add three **Scalar Parameters**:

| Name | Default | Description |
| :- | :- | :- |
| `DistortionStrength` | 0.01 | How far pixels shift |
| `DistortionScale` | 8.0 | Size of distortion waves |
| `DistortionSpeed` | 1.0 | Animation speed |

Add a **Time** node to drive the animation.



## 5. Main Custom Node – UV Distortion (Outputs Float2)

The Custom node will **not** sample the scene.  
It will return **distorted UVs**, which a SceneTexture node will sample.

### 5.1 Create the Custom Node

Add a **Custom** node:

- **Name:** `HeatDistortion`  
- **Output Type:** `Float2`  

Add these **Inputs** (exact spelling is critical):

| Input Name | Type | Connect From |
| :- | :- | :- |
| `uv` | Float2 | Output of `GetScreenUVs` |
| `timeValue` | Float1 | Time node |
| `distortionStrength` | Float1 | DistortionStrength |
| `distortionScale` | Float1 | DistortionScale |
| `distortionSpeed` | Float1 | DistortionSpeed |

Wire each input accordingly.

### 5.2 HLSL Code (No `main()`, No scene sampling)

Paste this HLSL into the **Code** field of the `HeatDistortion` Custom node:

```hlsl
// Inputs:
//   uv                 : float2 screen-space UV in [0,1]
//   timeValue          : time value from the Time node
//   distortionStrength : amplitude of UV displacement
//   distortionScale    : size / frequency of distortion waves
//   distortionSpeed    : animation speed multiplier

// Start with original UV
float2 baseUV = uv;

// Center UV for nicer waves
float2 centredUV = baseUV - float2(0.5, 0.5);

// Scale to control wave frequency
float2 p = centredUV * distortionScale;

// Animated time value
float t = timeValue * distortionSpeed;

// Two simple sine/cosine waves for distortion
float waveX = sin(p.y * 6.0 + t * 2.0);
float waveY = cos(p.x * 4.0 - t * 1.5);

// Build a flow vector
float2 flow = float2(waveX, waveY) * 0.5;

// Apply distortion strength
float2 offset = flow * distortionStrength;

// Final UVs
float2 distortedUV = baseUV + offset;

// Clamp to valid screen range
distortedUV = clamp(distortedUV, 0.0, 1.0);

// Return only the distorted UVs
return distortedUV;
```

## 6. Sample the Scene Using the Distorted UVs

1. Add a **SceneTexture** node.  
2. Set **Scene Texture Id → PostProcessInput0**.  
3. Connect:

   - `HeatDistortion` output (Float2) → **SceneTexture.UVs**  
   - **SceneTexture.Color** → **Emissive Color**

This is the correct Unreal workflow:

- HLSL node computes UVs  
- SceneTexture node fetches the scene at those UVs  



## 7. Assign the Material

1. Select the **Post Process Volume**.  
2. Under **Post Process Materials**, assign:

   ```text
   M_HeatDistortion_Simple
   ```

3. Set **Blend Weight = 1.0**.  

Press **Play**. You should now see shimmering heat waves.

If it’s too subtle, temporarily increase:

```text
DistortionStrength = 0.03
```



## 8. Student Tasks

### Task 1 — Create Desert and Underwater Variants

Make two Material Instances and experiment with:

| Effect | Strength | Scale | Speed |
| :- | :- | :- | :- |
| Desert shimmer | 0.005–0.015 | 6–12 | 0.6–1.2 |
| Underwater waves | 0.01–0.02 | 3–6 | 0.3–0.8 |

Describe which values feel more like:

- hot road / desert heat  
- underwater refraction  

### Task 2 — Fade Distortion Near the Top

Use the **Y** component of `ScreenUV` from `GetScreenUVs`:

1. Take the `ScreenUV` output.  
2. Use a **Component Mask** (G) or BreakFloat2 to get `ScreenUV.y`.  
3. Multiply `DistortionStrength` by `ScreenUV.y`.  
4. Feed this product into the `distortionStrength` input of the Custom node.

This makes distortion strongest near the bottom of the screen and weaker near the top.

### Task 3 — Combine Effects (Design Exercise)

Think about combining this heat distortion with:

- **Vignette** (Lab 7)  
- **Chromatic Aberration** (Lab 8)  
- **Depth Fog** (Lab 10)

## 9. Reflective Questions

1. Why does offsetting UVs and re-sampling the screen create a mirage effect?  
2. How do `DistortionScale` and `DistortionSpeed` change the appearance of the waves?  
3. Why must distorted UVs be clamped to the `[0,1]` range?  
4. What visual artefacts appear if `DistortionStrength` is set too high?  
5. Where might this effect enhance gameplay (e.g. signalling heat or danger), and where might it hinder the player?
