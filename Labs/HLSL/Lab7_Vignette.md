# Lab 7 – Screen-Space Vignette with Custom Nodes & HLSL  
## Module: 3D Game Development



## Overview

In this lab, you will implement a **screen-space vignette** effect as a **Post-Process Material** using a **Custom HLSL node** in Unreal Engine.

A vignette darkens (or tints) the edges of the screen, drawing the player’s eye toward the centre. This is a very common camera/post effect in modern games and is mathematically simple, making it ideal for early HLSL practice.

The vignette will be created by:

- computing the **distance from the screen centre** for each pixel,  
- using `smoothstep` to create a soft radial falloff,  
- blending this radial mask with the original scene colour.



## Learning Outcomes

By completing this lab, you will be able to:

- Create a **Post-Process Material** that modifies the final rendered image.  
- Use a **Custom HLSL node** to compute a radial mask based on UV coordinates.  
- Apply `smoothstep` to create a soft vignette transition.  
- Blend the vignette with the scene colour using an intensity parameter.  
- Expose vignette radius and softness as artist-friendly controls.



## 1. Create a Post Process Volume

1. Drag a **Post Process Volume** into your level.  
2. In the Details panel:  
   - Enable **Infinite Extent (Unbound)** → *true*.  
3. Under **Post Process Materials**, add an **Array Element**.  
   - Set the type to **Asset Reference**.  
4. You will assign the vignette material here later.



## 2. Create the Vignette Material

1. In the Content Browser, create a new Material:  

   ```text
   M_Vignette
   ```

2. Open the material and set:  
   - **Material Domain → Post Process**  

Only **Emissive Color** will be used as the output.



## 3. Helper Custom Node – Screen UVs

As in previous labs, you will use a tiny Custom node to obtain correct screen-space UVs for each pixel.

1. Add a **Custom** node:
   - **Name:** `GetScreenUVs`  
   - **Output Type:** `Float2`  

2. In the **Code** field, paste:

   ```hlsl
   return GetSceneTextureUV(Parameters);
   ```

This returns the current pixel’s screen-space UV in the range **[0, 1]**.



## 4. Add Vignette Parameters

Add the following **Scalar Parameters** to control the effect:

| Parameter Name      | Default | Description                           |
| :- | :- | :- |
| `VignetteRadius`    | `0.4`   | Radius where darkening begins        |
| `VignetteSoftness`  | `0.25`  | How softly the vignette fades out    |
| `VignetteIntensity` | `1.0`   | Strength of the vignette effect      |

You may group these parameters visually in the material instance under a **Vignette** group (optional, for organisation).



## 5. Main Custom Node – Vignette Logic

You will now create a **Custom HLSL node** that:

1. Reads the screen UV.  
2. Computes the radial distance from the centre.  
3. Uses `smoothstep` to calculate a vignette mask.  
4. Samples the scene colour.  
5. Blends the scene colour with the vignette using an intensity parameter.

### 5.1 Add the Custom Node

1. Add a new **Custom** node:
   - **Name:** `Vignette`  
   - **Output Type:** `Float3`  

2. In the **Inputs** list for this node, add:

| Input Name | Connect |
|:-|:-|
| `uv`                | Output of `GetScreenUVs`            |
| `vignetteRadius`    | `VignetteRadius` parameter          |
| `vignetteSoftness`  | `VignetteSoftness` parameter        |
| `vignetteIntensity` | `VignetteIntensity` parameter       |



### 5.2 HLSL Code

Paste the following **fully commented** HLSL code into the Custom node’s **Code** field:

```hlsl
// Inputs:
//   uv                : float2 screen-space UV (0..1)
//   vignetteRadius    : scalar radius where darkening begins
//   vignetteSoftness  : scalar width of the soft edge
//   vignetteIntensity : scalar intensity of the vignette effect

// Step 1: Move UV so that (0.5, 0.5) is treated as the origin (screen centre).
// After this, the centre is (0, 0) and corners are roughly (-0.5, -0.5) to (0.5, 0.5).
float2 centeredUV = uv - float2(0.5, 0.5);

// Step 2: Compute distance from the screen centre.
// This is the radius in screen space.
float dist = length(centeredUV);

// Step 3: Use smoothstep to create a radial falloff.
// We define an inner radius (vignetteRadius) and an outer radius
// (vignetteRadius + vignetteSoftness).
// - For dist <= vignetteRadius, the smoothstep output is ~0 (no darkening).
// - For dist >= vignetteRadius + vignetteSoftness, the output is ~1 (full darkening).
float edge = smoothstep(vignetteRadius, vignetteRadius + vignetteSoftness, dist);

// Step 4: Convert this "edge" into a vignette factor that is 1 at the centre
// and approaches 0 towards the edges.
// edge goes from 0 (centre) to 1 (edges), so we invert it.
float vignetteFactor = 1.0 - edge;

// Step 5: Sample the original scene colour from PostProcessInput0.
// 14 = PostProcessInput0 in Unreal post-process materials.
float3 sceneColor = SceneTextureLookup(uv, 14, false).rgb;

// Step 6: Compute a final darkening factor based on vignetteIntensity.
// - When vignetteIntensity = 0, factor should be 1.0 (no change).
// - When vignetteIntensity = 1, factor should be vignetteFactor (full vignette).
float finalFactor = lerp(1.0, vignetteFactor, vignetteIntensity);

// Step 7: Apply the vignette by multiplying the scene colour
// by the final factor. This darkens the edges more than the centre.
float3 finalColor = sceneColor * finalFactor;

// Step 8: Return the final vignetted scene colour.
return finalColor;
```



## 6. Connect Output to Emissive Color

Connect the **Vignette** Custom node output (Float3) → **Emissive Color**.

Your material now applies a vignette to the entire scene as a post-process effect.



## 7. Assign the Material to the Post Process Volume

1. Go back to your level and select the **Post Process Volume**.  
2. Under **Post Process Materials**, assign:

   ```text
   M_Vignette
   ```

3. Set **Blend Weight = 1.0**.

Press **Play** to see the vignette darkening the screen edges.



## 8. Student Tasks

### Task 1 – Tuning Radius and Softness

Use a **Material Instance** of `M_Vignette` and experiment with:

- `VignetteRadius` values between `0.2` and `0.6`.  
- `VignetteSoftness` values between `0.1` and `0.4`.  

Questions to consider:

- How does a small radius with high softness look compared to a larger radius with low softness?  
- Which settings feel more subtle vs more dramatic?



### Task 2 – Dynamic Vignette for Low Health (Design Exercise)

Imagine a game where the vignette becomes stronger as the player’s health decreases.

- How could you drive `VignetteIntensity` from a Blueprint (no need to implement fully, just outline the idea)?  
- What intensity values might you use for **full health** vs **critical health**?

### Task 3 – Experimental Colour Vignette (Optional Extension)

Extend the vignette to apply **colour tinting** to the edges:

1. Add a **Vector Parameter** called `VignetteTint` (default: black or dark red).  
2. Modify the HLSL so that:
   - the edges are tinted by `VignetteTint`,  
   - the centre remains close to the original scene colour.  

Hint: you could lerp between `sceneColor` and `VignetteTint * sceneColor` using the same `edge` or `1 - vignetteFactor`.

## 9. Reflective Questions

1. Why do we subtract `0.5` from the UVs before computing the distance?  
2. How does `smoothstep` help create a smooth transition for the vignette compared to a hard threshold?  
3. What artefacts might appear if the vignette radius or softness values are set poorly (for example, radius too close to 0 or 1)?  
4. How does changing `VignetteIntensity` differ visually from changing `VignetteRadius`?  
5. In what situations might a vignette improve player focus or readability, and in what situations might it be distracting?